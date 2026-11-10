# 02 — Application Backup and Restore with OADP (AWS S3)

**Date:** 10 November 2026
**Cluster:** OpenShift Container Platform 4.22.6, 3-node compact cluster
**Tooling:** OADP Operator 1.6.2 (Velero), AWS S3 as off-site backup storage

Commands run from the **bastion** unless stated otherwise.

## Objective

Protect an individual application (namespace) by backing it up to off-site object storage, then simulate accidental deletion and restore it.

## Concept

| | etcd backup (Runbook 01) | OADP backup (this runbook) |
|---|---|---|
| Scope | Entire cluster state | Selected namespaces / applications |
| Restore | Whole cluster back to one point in time | Only the chosen application |
| Use case | Disaster recovery of the control plane | Accidental deletion, app migration between clusters |
| Storage | Snapshot file copied off-node | S3-compatible object storage |

**Architecture:**

```
OADP Operator (manager)  →  reads DataProtectionApplication  →  deploys Velero
Velero (worker)          →  Backup / Restore objects          →  AWS S3 bucket (off-site)
```

---

## 1. AWS preparation (least privilege)

- Zero-spend budget alert enabled.
- Private S3 bucket created in `us-east-1` (Block Public Access on).
- Dedicated IAM user `oadp-backup` (programmatic access only) with a policy scoped to the single bucket:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ObjectAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::<bucket-name>/*"
    },
    {
      "Sid": "BucketList",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<bucket-name>"
    }
  ]
}
```

Access keys are used instead of an IAM role because the cluster runs on-premises (roles require an AWS-trusted identity such as EC2, EKS OIDC or OpenShift with STS).

---

## 2. Install OADP and configure S3

**Install the OADP Operator** from OperatorHub into the `openshift-adp` namespace, then verify:

```bash
oc get csv -n openshift-adp
# oadp-operator.v1.6.2   OADP Operator   1.6.2   Succeeded
```

**Store the AWS credentials as a secret:**

```bash
cat > ~/credentials-velero <<EOF
[default]
aws_access_key_id=<ACCESS_KEY_ID>
aws_secret_access_key=<SECRET_ACCESS_KEY>
EOF
chmod 600 ~/credentials-velero

oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=$HOME/credentials-velero
rm -f ~/credentials-velero
```

**Create the DataProtectionApplication:**

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-aws
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - openshift
        - aws
  backupImages: false
  backupLocations:
    - name: aws-s3
      velero:
        provider: aws
        default: true
        objectStorage:
          bucket: <bucket-name>
          prefix: ocp4
        config:
          region: us-east-1
          profile: default
        credential:
          name: cloud-credentials
          key: cloud
```

```bash
oc apply -f ~/dpa-aws.yaml
```

**Verify:**

```bash
oc get dpa -n openshift-adp
# dpa-aws   RECONCILED True

oc get bsl -n openshift-adp
# aws-s3   Available   true

oc get pods -n openshift-adp
# openshift-adp-controller-manager-...   Running
# velero-...                             Running
```

---

## 3. Deploy a sample application

```bash
oc new-project oadp-demo
oc new-app --name=webapp --image=registry.access.redhat.com/ubi9/httpd-24
oc create configmap webapp-config --from-literal=ENV=lab --from-literal=OWNER=anjala
oc expose service/webapp

oc get all,configmap,route -n oadp-demo
curl -s -o /dev/null -w "%{http_code}\n" http://webapp-oadp-demo.apps.ocp4.lab.local
# 403  (app reachable — default Apache page)
```

---

## 4. Backup

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: oadp-demo-backup-1
  namespace: openshift-adp
spec:
  includedNamespaces:
    - oadp-demo
  storageLocation: aws-s3
  ttl: 720h0m0s
```

```bash
oc apply -f ~/backup-oadp-demo.yaml
oc describe backup oadp-demo-backup-1 -n openshift-adp | grep -A5 -E "Phase|Items"
# Phase: Completed
# Items Backed Up: 45 / Total Items: 45
```

Backup stored in S3 under `ocp4/backups/oadp-demo-backup-1/`.

---

## 5. Simulate accidental deletion

```bash
oc delete project oadp-demo
oc get project oadp-demo
# NotFound
```

---

## 6. Restore

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: oadp-demo-restore-1
  namespace: openshift-adp
spec:
  backupName: oadp-demo-backup-1
  restorePVs: false
```

```bash
oc apply -f ~/restore-oadp-demo.yaml
oc get restore oadp-demo-restore-1 -n openshift-adp -o jsonpath='{.status.phase}{"\n"}'
# Completed
```

---

## 7. Verification

```bash
oc get all -n oadp-demo
# pod/webapp-...   1/1 Running
# deployment, service, route, imagestream restored

oc describe configmap webapp-config -n oadp-demo
# Data: ENV=lab, OWNER=anjala
# Labels: velero.io/backup-name=oadp-demo-backup-1
#         velero.io/restore-name=oadp-demo-restore-1

curl -s -o /dev/null -w "%{http_code}\n" http://webapp-oadp-demo.apps.ocp4.lab.local
# 403  (app reachable again)

oc describe restore oadp-demo-restore-1 -n openshift-adp | grep -A3 -E "Warnings|Errors"
# Warnings: 5 (non-critical), no errors
```

**Observation:** the Service received a new ClusterIP after restore (`172.30.166.30` → `172.30.191.175`). Velero does not restore cluster-assigned IPs, so applications should always reference services by DNS name.

---

## Result

- Application namespace backed up to off-site AWS S3 (45 objects)
- Namespace deleted and fully restored, including configuration data
- Application reachable through its route after restore
- Least-privilege IAM access for the backup user

## Reference

- [OpenShift 4.22 — OADP application backup and restore](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/backup_and_restore/oadp-application-backup-and-restore)
