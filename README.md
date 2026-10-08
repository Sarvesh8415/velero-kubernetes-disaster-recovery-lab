Velero Kubernetes Backup & Disaster Recovery Lab
1. Project Overview
This project demonstrates an end-to-end Kubernetes backup and
disaster-recovery workflow using Velero and an external MinIO
S3-compatible backup repository.

The lab was implemented on a local two-node Kubernetes cluster created
with k3d/k3s and Docker.

The project proves recovery in two disaster scenarios:
    1. Application namespace deletion and restore.
    2. Complete Kubernetes cluster deletion, recreation, and restore from
   the same external MinIO repository.

The recovery validation includes both Kubernetes resources and the
original persistent application data.

2. Objective
The main objectives are to:
    - Create a two-node local Kubernetes cluster with one control-plane
    node and one worker node.
    - Run MinIO outside the Kubernetes cluster as persistent external
    backup storage.
    - Deploy a test application containing a Deployment, Service,
    ConfigMap, Secret, and PVC.
    - Write unique data to the PVC.
    - Back up Kubernetes resources and persistent file-system data using
    Velero.
    - Restore the application after namespace deletion.
    - Delete the complete Kubernetes cluster and restore the workload into
    a newly created cluster.
    - Configure a daily Velero backup schedule with seven-day retention.
    - Document implementation, validation, troubleshooting, and recovery
    evidence.
3. Architecture
The architecture is available at:
    architecture/architecture.png
High-level flow:
    Original k3d Kubernetes Cluster
            |
            | Velero backup
            v
    External MinIO (S3-compatible)
            |
            | Velero restore
            v
    New k3d Kubernetes Cluster
MinIO remains outside the Kubernetes cluster so that deleting the
Kubernetes cluster does not delete the backup repository.


4. Main Technologies
  Component                 Technology

  Container runtime         Docker
  Kubernetes                k3d / k3s
  Kubernetes CLI            kubectl
  Backup/DR                 Velero
  Persistent-data backup    Velero Node Agent / file-system backup
  Object storage            MinIO
  Application               NGINX container
  Kubernetes manifests      YAML
  Documentation             Markdown
  Development environment   VS Code


5. Tool Versions
The lab was implemented with:

- Docker: 29.7.2
- kubectl: v1.37.1
- k3d: v5.9.0
- k3s: v1.35.5-k3s1
- Velero CLI: v1.18.4
- MinIO: cgr.dev/chainguard/minio:latest
Helm was not used.


6. Repository Structure

    velero-disaster-recovery-lab/
    ├── README.md
    ├── architecture/
    │   └── architecture.png
    ├── kubernetes/
    │   ├── namespace.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── configmap.yaml
    │   ├── secret-example.yaml
    │   └── pvc.yaml
    ├── velero/
    │   ├── installation-notes.md
    │   ├── backup-commands.md
    │   ├── restore-commands.md
    │   └── schedule.md
    ├── evidence/
    │   ├── before-backup/
    │   ├── after-delete/
    │   ├── after-restore/
    │   └── new-cluster-restore/
    └── report/
        └── velero-disaster-recovery-report.md

Local-only files such as Velero credentials and MinIO persistent data
must not be committed to Git.

7. Kubernetes Cluster
The cluster name used in the lab was:  velero-lab

It was created with one server/control-plane node and one worker node:
    k3d cluster create velero-lab --servers 1 --agents 1

Verification:
    kubectl get nodes -o wide
    kubectl config current-context
Both nodes were verified in Ready state.

8. External MinIO
MinIO was run outside Kubernetes in a Docker container and its data was
persisted on the host.

MinIO image troubleshooting

We initially tried the commonly used MinIO images:
    docker pull minio/minio:latest
and:
    docker pull quay.io/minio/minio:RELEASE.2025-09-06T17-38-46Z
but we encountered access/pull issues.
So we used the public Chainguard MinIO image:
    cgr.dev/chainguard/minio:latest
and the pull succeeded.

The MinIO container was named:
    velero-minio

The Velero backup bucket was:
velero

The MinIO data directory was persisted on the host under:
minio/data/

The backup endpoint used by Velero was:
http://host.docker.internal:9000

Credentials were stored locally in:
velero/credentials-velero

and excluded from Git.

9. Velero
Velero was installed in the velero namespace with the AWS
S3-compatible plugin and Node Agent/file-system backup enabled.

The installation used:
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.14.0 --bucket velero --secret-file .\velero\credentials-velero --use-volume-snapshots=false --use-node-agent --default-volumes-to-fs-backup --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://host.docker.internal:9000 --wait

The installer initially reported a timeout, but investigation showed
that the Velero server was running and the BackupStorageLocation became
available. No unnecessary reinstall was performed.
The Node Agent was then verified as running after correcting its
HostPath configuration.

10. Test Application
The application namespace was:
velero-lab

The application contains:
    - Namespace
    - Deployment
    - Service
    - ConfigMap
    - Secret
    - PersistentVolumeClaim
    - Pod mounting the PVC

The application container uses NGINX and mounts the PVC at:
/data

The persistent file created for the recovery test was:
/data/important-data.txt

Its original content was:

Velero Disaster Recovery Test
Intern: Sarvesh Patil
Record-ID: VELERO-DR-001
Created: 2026-10-08
This data must survive backup and disaster recovery.

11. Backups
Two manual backups were created.
Backup 01
    velero backup create velero-lab-backup-01 --include-namespaces velero-lab --default-volumes-to-fs-backup
    The backup completed successfully.
Backup 02
    velero backup create velero-lab-backup-02 --include-namespaces velero-lab --default-volumes-to-fs-backup
    The second backup was created after the Service was present and
    completed successfully with 13/13 items backed up.
    The backup data was stored in the external MinIO repository.

12. Disaster Recovery Test 1
The application namespace was deleted:
    kubectl delete namespace velero-lab

The namespace and application resources were verified as gone.
The application was restored from the backup and the following were
verified:
    - Namespace
    - Pod
    - Deployment
    - ReplicaSet
    - Service
    - ConfigMap
    - Secret
    - PVC
    - Persistent application data
The restored /data/important-data.txt content matched the original
content exactly.

13. Disaster Recovery Test 2
The original Kubernetes cluster was completely deleted:
k3d cluster delete velero-lab

MinIO remained running outside the cluster and the backup data remained
available.
    A fresh two-node cluster was created:
    k3d cluster create velero-lab --servers 1 --agents 1

Velero was installed again and connected to the same MinIO repository.
The old backups became visible:
    velero-lab-backup-01
    velero-lab-backup-02

The second backup was restored:
    velero restore create velero-lab-restore-02 --from-backup velero-lab-backup-02
The restore completed with:
    13/13 items restored
    0 errors
The application resources and PVC were verified in the newly created
cluster.

Finally, the persistent file was read from the restored application:
kubectl exec -n velero-lab <restored-pod-name> -- cat /data/important-data.txt
The content exactly matched the original data.

14. Scheduled Backup
A daily scheduled backup was configured:
velero schedule create velero-lab-daily --schedule "0 2 * * *" --include-namespaces velero-lab --default-volumes-to-fs-backup --ttl 168h

Configuration:
    - Schedule: Daily at 2:00 AM
    - Namespace: velero-lab
    - File-system backup: Enabled
    - Retention: 168 hours / 7 days

The schedule was verified with:
    velero schedule get
    velero schedule describe velero-lab-daily

15. Evidence
Evidence is organized under:
    evidence/
    ├── before-backup/
    ├── after-delete/
    ├── after-restore/
    └── new-cluster-restore/
The evidence includes command outputs and screenshots for:
    - Cluster state
    - MinIO
    - Velero
    - Node Agent troubleshooting
    - Backup completion
    - Persistent-volume backup
    - Namespace deletion
    - Namespace restore
    - Complete cluster deletion
    - MinIO survival
    - New cluster creation
    - Old backup discovery
    - New-cluster restore
    - Persistent-data recovery
    - Scheduled backup


16. Security
Do not commit:
    - MinIO access keys or secret keys
    - Velero credential files
    - kubeconfig credentials
    - Cloud credentials
    - Access tokens
    - Other secrets
The local credentials file is:
    velero/credentials-velero
and should remain ignored by Git.

17. Final Result
The disaster-recovery workflow was successfully demonstrated.

    BACKUP
    ↓
    NAMESPACE FAILURE
    ↓
    RESTORE
    ↓
    DATA VALIDATION

    and

    BACKUP
    ↓
    COMPLETE CLUSTER LOSS
    ↓
    NEW CLUSTER
    ↓
    RECONNECT TO MINIO
    ↓
    RESTORE
    ↓
    DATA VALIDATION

Both Kubernetes resources and the original persistent application data
were recovered successfully.

See report/velero-disaster-recovery-report.md for the detailed
technical report