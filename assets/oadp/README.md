# Process

## Deployment

## Pre-reqs

- Create the Bucket to store backup

```bash
export REGION=us-west-2
aws s3api create-bucket --bucket jparrill-oadp --region=$REGION --create-bucket-configuration LocationConstraint=$REGION
```

```json
{
    "Location": "http://jparrill-oadp.s3.amazonaws.com/"
}
```

- Create the IAM user for velero

```bash
aws iam create-user --user-name jparrill-velero
```

```json
{
    "User": {
        "Path": "/",
        "UserName": "jparrill-velero",
        "UserId": "xxxxxxxxxx",
        "Arn": "arn:aws:iam::820196288204:user/jparrill-velero",
        "CreateDate": "2024-03-20T10:51:49+00:00"
    }
}
```

- Create Velero policy file

```bash
export BUCKET=jparrill-oadp

cat > velero-policy.json <<EOF
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
                "arn:aws:s3:::${BUCKET}/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": [
                "arn:aws:s3:::${BUCKET}"
            ]
        }
    ]
}
EOF
```

- Attach the policy file to iam user

```bash
aws iam put-user-policy --user-name jparrill-velero --policy-name jparrill-velero --policy-document file://velero-policy.json
```

- Create an access key for Velero user

```bash
aws iam create-access-key --user-name jparrill-velero
```

```json
{
    "AccessKey": {
        "UserName": "jparrill-velero",
        "AccessKeyId": "<AWS_ACCESS_KEY_ID>",
        "Status": "Active",
        "SecretAccessKey": "<AWS_SECRET_ACCESS_KEY>",
        "CreateDate": "2024-03-20T10:56:35+00:00"
    }
}
```

- Create the creds file to be consumed by Velero

```bash
cat << EOF > ~/.aws/credentials-velero
[default]
aws_access_key_id=<AWS_ACCESS_KEY_ID>
aws_secret_access_key=<AWS_SECRET_ACCESS_KEY>
EOF
```

- Create the default secret

```bash

oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=${HOME}/.aws/credentials-velero
```

- Create the DataProtectionApplication (Openshift ADP Config)[https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/installing/installing-oadp-aws.html#oadp-installing-dpa-1-3_installing-oadp-aws]

```yaml
---
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-instance
  namespace: openshift-adp
spec:
  backupLocations:
    - name: default
      velero:
        provider: aws
        default: true
        objectStorage:
          bucket: jparrill-oadp
          prefix: objects
        config:
          region: us-west-2
          profile: "default"
        credential:
          key: cloud
          name: cloud-credentials
  snapshotLocations:
    - velero:
        provider: aws
        config:
          region: us-west-2
          profile: "default"
        credential:
          key: cloud
          name: cloud-credentials
  configuration:
    nodeAgent:
      enabled: true
      uploaderType: kopia
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
      resourceTimeout: 10m
```

- Check the BackupStorageLocations

```bash
oc get BackupStorageLocations -n openshift-adp
oc get dpa dpa-instance -n openshift-adp -o jsonpath='{.status}' | jq
```

