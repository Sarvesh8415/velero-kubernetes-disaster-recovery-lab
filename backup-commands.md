Velero Kubernetes Disaster Recovery Lab – Commands from Start to End
Format: Command → Purpose → What it returns

1. Verify Prerequisites
    Command: docker --version
    Purpose: Verify Docker installation.
    What it returns: Installed Docker version.

    Command: docker info
    Purpose: Verify Docker Desktop/engine is running.
    What it returns: Docker client/server information.

    Command: kubectl version --client
    Purpose: Verify kubectl installation.
    What it returns: kubectl client version.

    Command: k3d version
    Purpose: Verify k3d installation.
    What it returns: k3d version information.

    Command: kubectl kustomize version
    Purpose: Verify Kustomize version.
    What it returns: Kustomize version.

    Command: velero version
    Purpose: Verify Velero CLI installation.
    What it returns: Velero client version.

    Command: git --version
    Purpose: Verify Git installation.
    What it returns: Git version.

2. Create Kubernetes Cluster

    Command: k3d cluster create velero-lab --servers 1 --agents 1
    Purpose: Create a two-node local Kubernetes cluster.
    What it returns: Creates one server and one agent.

    Command: kubectl get nodes -o wide
    Purpose: Verify cluster nodes.
    What it returns: Returns both nodes and Ready status.

    Command: kubectl config current-context
    Purpose: Verify active Kubernetes context.
    What it returns: Returns k3d-velero-lab.

    Command: kubectl cluster-info
    Purpose: Verify Kubernetes connectivity.
    What it returns: Returns control-plane and cluster endpoints.

3. MinIO External Backup Storage

    Command: docker pull cgr.dev/chainguard/minio:latest
    Purpose: Pull a working MinIO image source used during the lab.
    What it returns: Docker pull result and image availability.

    Command: docker run -d --name velero-minio -p 9000:9000 -p 9001:9001 -e MINIO_ROOT_USER=minio -e MINIO_ROOT_PASSWORD=minio1234 -v "${PWD}\minio\data:/data" quay.io/minio/aistor/minio:RELEASE.2026-09-19T17-05-25Z
    Purpose: Run MinIO outside Kubernetes with persistent host storage.
    What it returns: Returns the MinIO container ID.

    Command: docker ps --filter "name=velero-minio"
    Purpose: Verify MinIO is running.
    What it returns: Returns the velero-minio container and ports.

    Command: Test-Path .\velero\credentials-velero
    Purpose: Verify the local Velero credentials file.
    What it returns: Returns True when the file exists.

4. Install Velero

    Command: velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.14.0 --bucket velero --secret-file .\velero\credentials-velero --use-volume-snapshots=false --use-node-agent --default-volumes-to-fs-backup --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://host.docker.internal:9000 --wait

    Purpose: Install Velero and connect it to MinIO.
    What it returns: Creates Velero components; the installer timed out in this lab while Velero later became healthy.

    Command: kubectl get pods -n velero -o wide
    Purpose: Check Velero Server and Node Agents.
    What it returns: Returns pod status and node placement.

    Command: kubectl get backupstoragelocation -n velero
    Purpose: Verify the MinIO backup location.
    What it returns: Returns the backup location and availability.

    Command: kubectl logs deploy/velero -n velero --tail=100
    Purpose: Troubleshoot/check Velero Server.
    What it returns: Returns recent Velero logs.

5. Fix Velero Node Agent

    Command: kubectl describe pod <node-agent-pod> -n velero
    Purpose: Investigate Node Agent startup failure.
    What it returns: Returns pod details, events, mounts and errors.

    Command: Test-Path .\velero\node-agent-path-patch.json
    Purpose: Verify the patch file.
    What it returns: Returns True when it exists.

    Command: kubectl patch daemonset node-agent -n velero --type=json --patch-file .\velero\node-agent-path-patch.json
    Purpose: Fix incorrect Linux HostPath values.
    What it returns: Returns patched DaemonSet confirmation.

    Command: kubectl get pods -n velero -o wide
    Purpose: Verify Node Agents after the fix.
    What it returns: Returns Node Agents as Running when successful.

6. Create Application Resources
    Command: kubectl apply -f .\kubernetes\namespace.yaml
    Purpose: Create the application namespace.
    What it returns: Creates velero-lab.

    Command: kubectl apply -f .\kubernetes\pvc.yaml
    Purpose: Create persistent storage request.
    What it returns: Creates app-data-pvc requesting 1Gi.

    Command: kubectl apply -f .\kubernetes\configmap.yaml
    Purpose: Create non-sensitive application configuration.
    What it returns: Creates app-config.

    Command: kubectl apply -f .\kubernetes\secret-example.yaml
    Purpose: Create application secret.
    What it returns: Creates app-secret.

    Command: kubectl apply -f .\kubernetes\deployment.yaml
    Purpose: Deploy the application and mount the PVC.
    What it returns: Creates Deployment and Pod.

    Command: kubectl apply -f .\kubernetes\service.yaml
    Purpose: Create application network Service.
    What it returns: Creates velero-dr-service.

    Command: kubectl get all -n velero-lab
    Purpose: Verify application resources.
    What it returns: Returns Pod, Service, Deployment and ReplicaSet.

    Command: kubectl get pvc -n velero-lab
    Purpose: Verify persistent storage.
    What it returns: Returns app-data-pvc and Bound status.

    Command: kubectl get configmap -n velero-lab
    Purpose: Verify ConfigMap.
    What it returns: Returns app-config.

    Command: kubectl get secret -n velero-lab
    Purpose: Verify Secret.
    What it returns: Returns app-secret.

7. Create and Verify Persistent Data

    Command: kubectl exec -n velero-lab <pod-name> -- sh -c "printf '%s\n' 'Velero Disaster Recovery Test' 'Intern: Sarvesh Patil' 'Record-ID: VELERO-DR-001' 'Created: 2026-10-08' 'This data must survive backup and disaster recovery.' > /data/important-data.txt"
    Purpose: Create unique test data in the PVC.
    What it returns: Creates important-data.txt.

    Command: kubectl exec -n velero-lab <pod-name> -- cat /data/important-data.txt
    Purpose: Verify persistent data.
    What it returns: Returns the file contents.

8. Create Velero Backups
    Command: velero backup create velero-lab-backup-01 --include-namespaces velero-lab --default-volumes-to-fs-backup
    Purpose: Create first backup.
    What it returns: Returns backup creation/status information.

    Command: velero backup get
    Purpose: List backups.
    What it returns: Returns backup names and phases.

    Command: velero backup describe velero-lab-backup-01 --details
    Purpose: Inspect first backup.
    What it returns: Returns backup details and file-system backup information.

    Command: velero backup logs velero-lab-backup-01
    Purpose: Inspect backup logs.
    What it returns: Returns backup logs; Windows retrieval may show the host.docker.internal resolution issue seen in the lab.

    Command: kubectl get podvolumebackups -n velero
    Purpose: Verify volume file-system backup.
    What it returns: Returns PodVolumeBackup resources and status.

    Command: velero backup create velero-lab-backup-02 --include-namespaces velero-lab --default-volumes-to-fs-backup
    Purpose: Create final DR backup.
    What it returns: Returns backup creation/status information.

    9. DR Test 1 – Namespace Delete and Restore
    Command: kubectl delete namespace velero-lab
    Purpose: Simulate application namespace loss.
    What it returns: Deletes velero-lab and its resources.

    Command: kubectl get namespace velero-lab
    Purpose: Verify deletion.
    What it returns: Returns NotFound after deletion.

    Command: kubectl get all -n velero-lab
    Purpose: Verify application resources are gone.
    What it returns: Returns no resources found.

    Command: velero restore create velero-lab-restore-01 --from-backup velero-lab-backup-02
    Purpose: Restore application namespace from backup.
    What it returns: Creates the restore.

    Command: velero restore get
    Purpose: Check restore status.
    What it returns: Returns restore phase.

    Command: velero restore describe velero-lab-restore-01 --details
    Purpose: Inspect restore details.
    What it returns: Returns restored item counts and volume restore information.

    Command: kubectl get all -n velero-lab
    Purpose: Verify restored application.
    What it returns: Returns restored resources.

    Command: kubectl get pvc -n velero-lab
    Purpose: Verify restored PVC.
    What it returns: Returns app-data-pvc as Bound.

    Command: kubectl exec -n velero-lab deploy/velero-dr-app -- cat /data/important-data.txt
    Purpose: Verify restored persistent data.
    What it returns: Returns the original file contents.

10. DR Test 2 – Full Cluster Delete and Re-create
    Command: k3d cluster list
    Purpose: Record the original cluster.
    What it returns: Returns the velero-lab cluster.

    Command: docker ps --filter "name=velero-minio"
    Purpose: Verify external MinIO is running.
    What it returns: Returns velero-minio.

    Command: k3d cluster delete velero-lab
    Purpose: Simulate complete cluster loss.
    What it returns: Deletes the k3d cluster.

    Command: k3d cluster list
    Purpose: Verify cluster deletion.
    What it returns: Returns no velero-lab cluster.

    Command: docker ps --filter "name=velero-minio"
    Purpose: Verify MinIO survived cluster deletion.
    What it returns: Returns MinIO still running.

    Command: k3d cluster create velero-lab --servers 1 --agents 1
    Purpose: Create a fresh two-node cluster.
    What it returns: Creates a new server and agent.

    Command: kubectl get nodes -o wide
    Purpose: Verify new cluster.
    What it returns: Returns both new nodes as Ready.

    Command: velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.14.0 --bucket velero --secret-file .\velero\credentials-velero --use-volume-snapshots=false --use-node-agent --default-volumes-to-fs-backup --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://host.docker.internal:9000 --wait
    Purpose: Reconnect Velero to the same external MinIO.
    What it returns: Installs Velero; old backups become discoverable after BSL validation.

    Command: kubectl get backupstoragelocation -n velero
    Purpose: Verify external backup location.
    What it returns: Returns Available when connected.

    Command: velero backup get
    Purpose: Verify old backups are visible.
    What it returns: Returns backup-01 and backup-02.

    Command: kubectl patch daemonset node-agent -n velero --type=json --patch-file .\velero\node-agent-path-patch.json
    Purpose: Fix Node Agent paths on the new cluster.
    What it returns: Returns patched DaemonSet confirmation.

    Command: kubectl get pods -n velero -o wide
    Purpose: Verify Velero components.
    What it returns: Returns Server and Node Agents as Running when healthy.

    Command: velero restore create velero-lab-restore-02 --from-backup velero-lab-backup-02
    Purpose: Restore into the new cluster.
    What it returns: Creates the second restore.

    Command: velero restore get
    Purpose: Verify second restore.
    What it returns: Returns restore phase.

    Command: kubectl get all -n velero-lab
    Purpose: Verify restored application.
    What it returns: Returns restored resources.

    Command: kubectl get pvc -n velero-lab
    Purpose: Verify restored PVC.
    What it returns: Returns Bound PVC.

    Command: kubectl exec -n velero-lab deploy/velero-dr-app -- cat /data/important-data.txt
    Purpose: Verify data survived complete cluster loss.
    What it returns: Returns original file contents.


11. Schedule Daily Backups
    Command: velero schedule create velero-lab-daily --schedule "0 2 * * *" --include-namespaces velero-lab --default-volumes-to-fs-backup --ttl 168h
    Purpose: Create daily backup with 7-day retention.
    What it returns: Creates velero-lab-daily.

    Command: velero schedule get
    Purpose: Verify schedule.
    What it returns: Returns schedule name, cron expression and status.

    Command: velero schedule describe velero-lab-daily
    Purpose: Inspect schedule configuration.
    What it returns: Returns cron expression, namespace, file-system backup and TTL.

12. Git Version Control

    Command: git init
    Purpose: Initialize local repository.
    What it returns: Creates .git.

    Command: git status
    Purpose: Check repository state.
    What it returns: Returns staged/untracked/working-tree status.


    Command: git add .
    Purpose: Stage project files.
    What it returns: Stages non-ignored files.

    Command: git commit -m "initial commit1"
    Purpose: Create Git commit.
    What it returns: Returns commit information or nothing to commit if already committed.

    Command: git remote -v

    Purpose: Verify GitHub remote.
    What it returns: Returns origin fetch/push URLs.
    Command: git push -u origin main

    Purpose: Push main to GitHub and set upstream.
    What it returns: Returns push result and tracking setup.

 13. General Troubleshooting / Verification
    Command: kubectl get namespace
    Purpose: List namespaces.
    What it returns: Returns system namespaces plus velero and velero-lab.

    Command: kubectl get pods -A
    Purpose: List Pods in all namespaces.
    What it returns: Returns Pods across the cluster.

    Command: kubectl get all -n velero
    Purpose: View Velero resources.
    What it returns: Returns Velero resources.

    Command: kubectl get all -n velero-lab
    Purpose: View application resources.
    What it returns: Returns application resources.

    Command: kubectl get nodes -o wide
    Purpose: Check node health.
    What it returns: Returns node details and status.

    Command: kubectl describe pod <pod-name> -n velero
    Purpose: Troubleshoot a Velero Pod.
    What it returns: Returns detailed Pod events/configuration.

    Command: kubectl logs deploy/velero -n velero --tail=100
    Purpose: Troubleshoot Velero Server.
    What it returns: Returns recent server logs.
