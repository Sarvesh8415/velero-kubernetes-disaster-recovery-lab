Velero Kubernetes Backup & Disaster Recovery Report

1. Introduction
This report documents the implementation and validation of a Kubernetes
backup and disaster-recovery lab using Velero and an external MinIO
S3-compatible object-storage repository.
The lab was implemented on a local two-node Kubernetes cluster using
k3d/k3s and Docker. The main objective was to prove that both Kubernetes
resources and persistent application data could be recovered after
application namespace loss and after complete Kubernetes cluster loss.

 

2. Objectives
The implementation objectives were:
    - Create a local Kubernetes cluster with one control-plane node and
    one worker node.
    - Run MinIO outside Kubernetes as persistent external backup storage.
    - Configure Velero to use MinIO as an S3-compatible backup repository.
    - Enable Velero Node Agent/file-system backup for persistent data.
    - Deploy a test application with a PVC.
    - Create unique persistent application data.
    - Perform and validate a Velero backup.
    - Simulate namespace-level application loss.
    - Restore the namespace and verify the original data.
    - Simulate complete cluster loss.
    - Recreate the cluster and reconnect Velero to the existing MinIO
    repository.
    - Discover and restore the previous backup.
    - Verify the application and exact persistent data.
    - Configure a daily scheduled backup with seven-day retention.
    - Document troubleshooting and evidence.

3. Architecture
The architecture is represented in:
architecture/architecture.png
The high-level architecture is:

                         External MinIO
                    S3-Compatible Repository
                         Bucket: velero
                              ^
                              |
                         Backup / Restore
                              |
        +---------------------+----------------------+
        |                                            |
        |                                            |
Original k3d Cluster                         New k3d Cluster
1 Control Plane + 1 Worker                  1 Control Plane + 1 Worker
        |                                            |
        |                                            |
   Velero Server                               Velero Server
   Node Agent                                  Node Agent
        |                                            |
   velero-lab Namespace                       velero-lab Namespace
        |                                            |
 Application + PVC                            Restored Application + PVC
        |                                            |
 important-data.txt                           important-data.txt

MinIO was deliberately kept outside the Kubernetes cluster so that
deleting the Kubernetes cluster would not remove the backup repository.


4. Prerequisites and Tool Versions
The following tools were used:
  Tool                 Version / Configuration
  Docker               29.7.2
  kubectl              v1.37.1
  k3d                  v5.9.0
  k3s                  v1.35.5-k3s1
  Velero CLI           v1.18.4
  MinIO                cgr.dev/chainguard/minio:latest
  IDE                  Visual Studio Code
  Kubernetes cluster   k3d, 1 server + 1 agent
Helm was not used.


5. Repository Structure
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
Local credentials and persistent MinIO data are not intended to be
committed to Git.

6. Kubernetes Cluster Creation
A two-node k3d cluster named velero-lab was created with one
control-plane/server node and one worker/agent node.
Command:
    k3d cluster create velero-lab --servers 1 --agents 1

    The cluster was verified using:
    kubectl get nodes -o wide
    The cluster contained:
    k3d-velero-lab-server-0    Ready    control-plane
    k3d-velero-lab-agent-0     Ready    <none>
    

7. External MinIO Deployment
MinIO was deployed separately from Kubernetes using Docker. Its storage
was persisted on the host under the project's minio/data/ directory.

7.1 MinIO Image Pull Troubleshooting

We initially tried the commonly used MinIO images:
docker pull minio/minio:latest
but we encountered access/pull issues.

So we used the public Chainguard MinIO image:
docker pull cgr.dev/chainguard/minio:latest

and the pull succeeded.
This working image was then used for the MinIO container.

7.2 MinIO Configuration
The MinIO container was named:
velero-minio
The S3-compatible bucket used by Velero was:
velero
The endpoint configured for Velero was:

docker run -d --name velero-minio -p 9000:9000 -p 9001:9001 
-e MINIO_ROOT_USER=minio -e MINIO_ROOT_PASSWORD=minio1234 
-v "${PWD}\minio\data:/data"  

http://host.docker.internal:9000
 

8. Velero Installation
Velero CLI version:
v1.18.4

velero/credentials-velero
[default]
aws_access_key_id=minio
aws_secret_access_key=minio1234


Velero was installed using the AWS plugin for S3-compatible storage,
with Node Agent/file-system backup enabled.
Command:

velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.14.0 --bucket velero --secret-file .\velero\credentials-velero --use-volume-snapshots=false --use-node-agent --default-volumes-to-fs-backup --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://host.docker.internal:9000 --wait
 

9. Node Agent Troubleshooting
 
kubectl get pods -n velero -o wide

10. Test Application

The application namespace was:
velero-lab

The application consisted of:
- Namespace
- Deployment
- Service
- ConfigMap
- Secret
- PersistentVolumeClaim
- Pod mounting the PVC

kubectl apply -f .\kubernetes\namespace.yaml
output: namespace/velero-lab created


11. Persistent Test Data

The following unique file was created:
/data/important-data.txt
Original content:
Velero Disaster Recovery Test
Intern: Sarvesh Patil
Record-ID: VELERO-DR-001
Created: 2026-10-08
This data must survive backup and disaster recovery.
The file was verified before backup.
This exact content was later compared against the restored file.

12. Pre-Backup Validation
Before the backup, the following resources and data were verified:
kubectl get all -n velero-lab
kubectl get pvc -n velero-lab
kubectl get configmap -n velero-lab
kubectl get secret -n velero-lab

kubectl exec -n velero-lab <pod-name> -- cat /data/important-data.txt


13. Velero Backup 01
The first backup was created using:
velero backup create velero-lab-backup-01 --include-namespaces velero-lab --default-volumes-to-fs-backup
 

14. Velero Backup 02
After the Service was present, a second backup was created:
velero backup create velero-lab-backup-02 --include-namespaces velero-lab --default-volumes-to-fs-backup
 

15. Disaster Recovery Test 1 --- Namespace Deletion
 
kubectl delete namespace velero-lab
 
16. Disaster Recovery Test 1 --- Restore
The application was restored using the Velero backup.
 velero restore create velero-lab-restore-01 --from-backup velero-lab-backup-02

The restored environment was validated using:
kubectl get all -n velero-lab

it recovered:
- Namespace
- Pod
- Deployment
- ReplicaSet
- Service
- ConfigMap
- Secret
- PVC
The restored PVC was Bound.

17. Disaster Recovery Test 1 --- Persistent Data Validation
 check this important-data.txt data is restored or not

18. Disaster Recovery Test 2 --- Complete Cluster Loss
After successful namespace-level recovery, the original cluster was
completely deleted.
Command:
k3d cluster delete velero-lab
 
cluster delete but MinIO remained running because it was outside the Kubernetes cluster.


20. New Kubernetes Cluster
 
k3d cluster create velero-lab --servers 1 --agents 1
 
 21. verify velero pods
    kubectl get pods -n velero -o wide

 

22. Disaster Recovery Test 2 --- Restore
The existing backup was restored into the new cluster:
velero restore create velero-lab-restore-02 --from-backup velero-lab-backup-02
.

23. Disaster Recovery Test 2 --- Resource Validation
The restored application was verified using:
kubectl get all -n velero-lab
kubectl get pvc -n velero-lab
kubectl get configmap -n velero-lab
kubectl get secret -n velero-lab
kubectl get namespace velero-lab
 

24. Disaster Recovery Test 2 --- Persistent Data Validation
The most important recovery validation was performed using:
kubectl exec -n velero-lab velero-dr-app-7685b6c6cb-rr5tt -- cat /data/important-data.txt
 
it verify : 
The application and its original persistent data were recovered after
the original Kubernetes cluster no longer existed.

 
25. Scheduled Backup
After both disaster-recovery tests were completed, a daily backup
schedule was created.
Command:
velero schedule create velero-lab-daily --schedule "0 2 * * *" --include-namespaces velero-lab --default-volumes-to-fs-backup --ttl 168h
Configuration:
  Setting              Value
  Schedule             0 2 * * *
  Frequency            Daily
  Namespace            velero-lab
  File-system backup   Enabled
  Retention            168 hours
  Retention in days    7
  Paused               False
 



26. Evidence Organization
Evidence was organized into four main directories:
evidence/
├── before-backup/
├── after-delete/
├── after-restore/
└── new-cluster-restore/
Before backup
Evidence includes:
- Application resources
- PVC
- ConfigMap
- Secret
- Persistent-data content
- Backup details
- PodVolumeBackup status
- MinIO backup contents
After namespace deletion
Evidence includes:
- Namespace deletion
- Resource absence
After namespace restore
Evidence includes:
- Restore status
- Restored resources
- PVC
- ConfigMap
- Secret
- Persistent data
New cluster recovery
Evidence includes:
- Original cluster nodes/context
- Cluster deletion
- MinIO surviving cluster deletion
- New cluster creation
- New cluster nodes/context
- Node Agent troubleshooting
- Old backup visibility
- Restore status/details
- Restored resources
- Restored PVC
- Restored persistent data
- Scheduled backup

  

27. Final Result
demonstrated a complete disaster-recovery
 
The final recovery flow was:
APPLICATION
    |
    v
VELERO BACKUP
    |
    v
EXTERNAL MINIO
    |
    +------------------------------+
    |                              |
    v                              v
NAMESPACE LOSS                COMPLETE CLUSTER LOSS
    |                              |
    v                              v
VELERO RESTORE                NEW KUBERNETES CLUSTER
    |                              |
    v                              v
RESOURCE + DATA              RECONNECT VELERO TO MINIO
RECOVERY                         |
                                 v
                            VELERO RESTORE
                                 |
                                 v
                          RESOURCE + DATA
                             RECOVERY
 

 