# ROSA with Velero

This article describes how to integrate ROSA with Velero for backing up and restoring all metadata stored in etcd for user-managed projects. This is a separate instance of Velero from that used by RedHat SRE. This integration will leverage short-term credentials and federated identify using the ROSA OIDC provider. Doing so avoids needing to create a dedicated user in IAM and exposing long-term credentials to Velero.

Important disclaimer: due to a known limitation with Restic (https://github.com/vmware-tanzu/velero/issues/2958) it is currently not possible to restore dynamically provisioned persistent volumes based on EFS. Only the metadata describing persistent volumes and persistent volume claims can be backed up and restored using Velero. To backup (and restore) the data contained within an EFS file system please use AWS Backup which integrates natively with EFS.

ROSA can be deployed as either a public or private cluster in STS mode as per these instructions:

https://mobb.ninja/docs/rosa/sts/

Do not install the Velero operator (OADP) from Operator Hub as the basic install method does not support STS operation. Instead we will use Helm to install Velero and Restic for backing up persistent volumes.

Create an S3 bucket in AWS that will be used as the target for all user-managed backups.

	aws s3api create-bucket --bucket velero-`uuid` --region ap-southeast-1 --create-bucket-configuration LocationConstraint=ap-southeast-1

Disable public access for this bucket from the AWS web console.

Create an IAM policy named velero-s3-policy with the following permissions. Substitute with the name of your S3 bucket accordingly.

	{
	    "Version": "2012-10-17",
	    "Statement": [
	       {
	            "Effect": "Allow",
	            "Action": [
	                "ec2:DescribeVolumes",
	                "ec2:DescribeSnapshots",
	                "ec2:CreateTags",
	                "ec2:CreateVolume",
	                "ec2:CreateSnapshot",
	                "ec2:DeleteSnapshot"
	            ],
	            "Resource": "*"
	        },			
	        {
	            "Effect": "Allow",
	            "Action": [
	                "s3:GetObject",
	                "s3:DeleteObject",
	                "s3:PutObject",
	                "s3:AbortMultipartUpload",
	                "s3:ListMultipartUploadParts"
	            ],
	            "Resource": [
	                "arn:aws:s3:::<your S3 bucket>/*"
	            ]
	        },
	        {
	            "Effect": "Allow",
	            "Action": [
	                "s3:ListBucket"
	            ],
	            "Resource": [
	                "arn:aws:s3:::<your S3 bucket>"
	            ]
	        }
	    ]
	}

Create an IAM role named velero-s3-irsa with the permissions defined by velero-S3-policy along with the following trust relationship. Substitute with the name of your OIDC provider accordingly. Importantly use the name of a service account that is not already installed in the cluster.

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
	        "Federated": "arn:aws:iam::<your AWS account>:oidc-provider/<your OIDC provider>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
	        "StringEquals": {
	          "<your OIDC provider>:sub": "system:serviceaccount:velero:velero-server"
	        }
	      }
	    }
	  ]
	}

You can obtain the identity of your OIDC provider using the rosa describe cluster command.

Prepare a Helm values.yaml file with the following contents. Note that if you plan to use dynamically created persistent volumes backed by EFS you need to set the global setting of defaultVolumesToRestic to false. Subsequently you will need to enable restic on a per-backup basis.

	configuration:
	  provider: aws
	  backupStorageLocation:
	    bucket: <your S3 bucket>
	    config:
	      region: <your AWS region>
	  volumeSnapshotLocation:
	    config:
	      region: <your AWS region>
	#
	serviceAccount:
	  server:
	    create: true
	    name: velero-server
	    annotations:
	      eks.amazonaws.com/role-arn: "arn:aws:iam::<your AWS account>:role/velero-s3-irsa"
	#
	initContainers:
	  - name: velero-plugin-for-aws
	    image: velero/velero-plugin-for-aws:v1.3.0
	    imagePullPolicy: IfNotPresent
	    volumeMounts:
	      - mountPath: /target
		name: plugins
	#
	credentials:
	  useSecret: false

Use Helm to install the latest version of Velero.

	helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
	helm repo update
	helm upgrade -i velero vmware-tanzu/velero --namespace velero --create-namespace -f values.yaml

Run the following commands to verify the setup.

	oc adm policy who-can use scc privileged
	oc get pods -o wide -n velero
	oc logs <velero pod> | grep "Backup storage location valid, marking as available"

***

The following scenario simulate the loss and restoration of the application and persistent volume.

Download and install the Velero CLI. Create a backup of an application namespace to be deleted and restored. Ideally this namespace should have a pod running with a persistent volume based on EFS.

	velero backup create my-backup-1 --include-namespaces my-project --wait

Delete the namespace/project and the underlying persistent volumes (or the PVC will not be terminated and the namespace not deleted). If you do not delete the persistent volumes the restore operation will not work. However before deleting the PV verify that the reclaim policy is set to Retain so that the delete operation will not cascade through to the underlying storage system.

Restore the configuration from the backup using Velero.

	velero restore create --from-backup my-backup-1 --wait

Note that this restores the pod, persistent volume and persistent volume claim and binds these to the original EFS access point.

***

Velero can also be used to restore from a total cluster failure scenario. In brief, the steps are:

1. Build a new ROSA cluster (either in a new VPC or existing VPC).
2. Switch the EFS fileystem mountpoint target to the ROSA VPC and create a new security group if a new VPC was created (see instructions in the next step).
3. Install the AWS EFS CSI Driver for ROSA as per https://github.com/redhat-apac-stp/rosa-with-aws-efs/blob/main/README.md
4. Install Velero as per the instructions above.
5. Verify access to the backup (oc get backup -n velero).
6. Verify the storageclass for EFS is loaded (oc describe sc/efs-sc) and that it is referencing the EFS fileystem.
7. Run the Velero restore command as per above.
8. Verify the pod, persistent volume and persistent volume claims are restored.

***

If you experience any errors or have questions please contact the author at jwilms@redhat.com

