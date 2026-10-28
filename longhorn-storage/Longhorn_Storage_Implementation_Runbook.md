# Longhorn Distributed Storage

# Implementation Runbook | k1 Kubernetes Lab

Target: Kubernetes v1.32.8 • Longhorn v1.10.2 • RHEL workers • VMware ESXi • Two storage replicas

## 1. Purpose and target design

Deploy Longhorn on three Kubernetes worker nodes to provide dynamically provisioned, replicated ReadWriteOnce volumes for a single PostgreSQL instance and a single JFrog Artifactory instance. The three control-plane nodes are excluded from Longhorn replica storage.

| Role | Nodes | Storage |
| --- | --- | --- |
| Control plane | k1-master1, k1-master2, k1-master3 | No Longhorn replica disk |
| Workers | k1-worker1, k1-worker2, k1-worker3 | Dedicated LVM-backed filesystem at /var/lib/longhorn |
| Storage policy | Longhorn V1 engine | 2 replicas on different workers; Retain PV reclaim policy |

Availability boundary: if these VMs share one physical ESXi host/datastore, Longhorn does not protect against loss of that host/datastore. Single-instance PostgreSQL and Artifactory will have restart/failover downtime.

## 2. Prerequisites and preflight

Run kubectl/Helm commands from an administrative host. Run host commands as root or with sudo on each worker. Confirm the Kubernetes cluster is healthy and that Helm is installed.

```bash
kubectl version
kubectl get nodes -o wide
helm version
kubectl get storageclass
```

Ensure worker DNS resolution, pod networking, sufficient free CPU/RAM, supported kernel, iSCSI tooling, and open inter-node Longhorn traffic. Validate version compatibility against Longhorn 1.10 documentation before deployment.

## 3. Prepare dedicated VMware disk and LVM on EACH worker

Attach a dedicated virtual disk to k1-worker1, k1-worker2 and k1-worker3 (example: 200–250 GiB each). Commands below assume the NEW EMPTY disk is /dev/sdb. Verify carefully: pvcreate and mkfs destroy data on the selected device.

lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
pvs; vgs; lvs

```bash
sudo pvcreate /dev/sdb
sudo vgcreate vg_longhorn /dev/sdb
sudo lvcreate -l 100%FREE -n lv_longhorn vg_longhorn
sudo mkfs.xfs /dev/vg_longhorn/lv_longhorn
sudo mkdir -p /var/lib/longhorn
```

Persist the mount using the filesystem UUID, not the volatile /dev/sdX name:

```bash
sudo blkid /dev/vg_longhorn/lv_longhorn
# Add to /etc/fstab (replace UUID):
UUID=<ACTUAL-UUID> /var/lib/longhorn xfs defaults 0 0
sudo mount -a
findmnt /var/lib/longhorn
df -hT /var/lib/longhorn
```

STOP if findmnt does not show the dedicated LV mounted at /var/lib/longhorn. Do not allow Longhorn to use an unmounted directory on the OS filesystem.

## 4. Install RHEL prerequisites on EACH worker

```bash
sudo dnf install -y iscsi-initiator-utils nfs-utils cryptsetup util-linux
sudo systemctl enable --now iscsid
sudo modprobe iscsi_tcp
echo iscsi_tcp | sudo tee /etc/modules-load.d/longhorn.conf
systemctl is-active iscsid
lsmod | grep iscsi_tcp
findmnt -o TARGET,PROPAGATION /
```

If subscription or repository errors occur, fix Red Hat repository/CA configuration before proceeding. Review SELinux and firewall requirements in the official Longhorn 1.10 installation documentation; do not disable security controls indiscriminately.

## 5. Label the storage workers

Use the exact Kubernetes node names below. The Longhorn Helm setting will create default disks only on labeled nodes.

```bash
for n in k1-worker1 k1-worker2 k1-worker3; do
  kubectl label node "$n" node.longhorn.io/create-default-disk=true --overwrite
done
kubectl get nodes -L node.longhorn.io/create-default-disk
```

## 6. Install Longhorn v1.10.2

Add the chart repository and create a values file. The chart is pinned to the version already validated in the other lab; recheck compatibility if the k1 cluster version differs.

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm show chart longhorn/longhorn --version 1.10.2
```

File: longhorn-values.yaml

```yaml
defaultSettings:
  defaultDataPath: /var/lib/longhorn
  createDefaultDiskLabeledNodes: true
  replicaSoftAntiAffinity: false

persistence:
  defaultClass: false
  defaultClassReplicaCount: 2
  defaultFsType: ext4
  reclaimPolicy: Retain
```

```bash
helm upgrade --install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --version 1.10.2 -f longhorn-values.yaml \
  --wait --timeout 15m
```

The settings above select the filesystem-backed replica data path and disable automatic default StorageClass selection. If this is an existing installation, review the Helm diff and storage state before upgrading.

## 7. Validate Longhorn components and disk paths

```bash
helm list -n longhorn-system
kubectl get pods -n longhorn-system -o wide
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get csidrivers
kubectl get storageclass
```

```bash
for n in k1-worker1 k1-worker2 k1-worker3; do
  echo "===== $n ====="
  kubectl get nodes.longhorn.io "$n" -n longhorn-system -o json | jq ".spec.disks"
done
```

Acceptance: all Longhorn managers/CSI pods are healthy; three worker Longhorn nodes are Ready/Schedulable; disk path is /var/lib/longhorn and disk capacity matches the dedicated LVM filesystem. Ensure no master is configured for replica scheduling.

## 8. StorageClass configuration

The Helm chart may already create a StorageClass named longhorn. Inspect it first; do not create a duplicate or delete an in-use class.

```bash
kubectl get sc longhorn -o yaml
kubectl get settings.longhorn.io default-replica-count -n longhorn-system -o jsonpath="{.value}{'\n'}"
```

Required effective StorageClass properties: provisioner driver.longhorn.io, numberOfReplicas "2", dataEngine "v1", fsType ext4, reclaimPolicy Retain, allowVolumeExpansion true. An explicit StorageClass replica count of 2 takes precedence over Longhorn’s global default of 3 for volumes created with that class.

If the chart-generated class does not match, create a separate class named longhorn-ssd rather than attempting to edit immutable parameters:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-ssd
provisioner: driver.longhorn.io
allowVolumeExpansion: true
reclaimPolicy: Retain
volumeBindingMode: Immediate
parameters:
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
  fsType: "ext4"
  dataLocality: "disabled"
  dataEngine: "v1"
```

In the remaining steps use storageClassName: longhorn if the existing class matches, otherwise longhorn-ssd.

## 9. Functional PVC and read/write test

kubectl create namespace longhorn-test

File: longhorn-test.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-test-pvc
  namespace: longhorn-test
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-test-pod
  namespace: longhorn-test
spec:
  containers:
  - name: test
    image: registry.access.redhat.com/ubi9/ubi-minimal:latest
    command: ["/bin/sh", "-c"]
    args: ["echo longhorn-ok > /data/test.txt; sleep infinity"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: longhorn-test-pvc
```

```bash
kubectl apply -f longhorn-test.yaml
kubectl get pvc,pods -n longhorn-test -o wide
kubectl exec -n longhorn-test longhorn-test-pod -- cat /data/test.txt
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get replicas.longhorn.io -n longhorn-system -o wide
```

Acceptance: PVC Bound; pod Running; test file reads longhorn-ok; volume Attached/Healthy; exactly two healthy replicas are on different worker nodes. Inspect pod events and Longhorn manager logs if any check fails.

## 10. Failure and recovery validation

Use only the test workload. First validate a planned drain of the worker hosting the test pod, then separately test an unexpected VM shutdown in a maintenance window. A hard shutdown can require node eviction and safe volume recovery; do not assume immediate failover.

```bash
kubectl get pod longhorn-test-pod -n longhorn-test -o wide
# Replace <POD_NODE> with actual node hosting the test pod:
kubectl drain <POD_NODE> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -n longhorn-test -o wide --watch
# After test:
kubectl uncordon <POD_NODE>
```

Confirm the file is intact after rescheduling, and that Longhorn rebuilds to two healthy replicas. Do not shut down two worker VMs simultaneously when only two replicas are configured.

## 11. PostgreSQL and Artifactory migration checklist

| Workload | Target PVC | Important considerations |
| --- | --- | --- |
| PostgreSQL (1 instance) | 50 GiB, ReadWriteOnce, longhorn | Take verified database backup; stop writes; restore/copy consistently; validate database startup and integrity. |
| JFrog Artifactory (1 instance) | 100 GiB or workload-based size, ReadWriteOnce, longhorn | Back up filestore/config; quiesce application; migrate data and permissions; validate repository downloads/uploads. |

Existing PVCs cannot simply change storageClassName in place. Provision replacement PVCs and perform a controlled backup/restore or data migration. Longhorn replication is not a backup: configure independent backups stored outside the ESXi datastore, and periodically test restoration.

## 12. Operations and troubleshooting

```bash
kubectl get pods -n longhorn-system
kubectl get nodes.longhorn.io -n longhorn-system
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get replicas.longhorn.io -n longhorn-system -o wide
kubectl describe pvc <PVC_NAME> -n <NAMESPACE>
kubectl get events -n <NAMESPACE> --sort-by=.lastTimestamp
kubectl logs -n longhorn-system -l app=longhorn-manager --tail=100 --prefix=true
```

Monitor available disk capacity, replica health, rebuild progress, node disk pressure, filesystem mounts, iSCSI status, and VMware datastore free space. Preserve free capacity for replica rebuilds and snapshots.

## 13. Completion checklist

- [ ] Three workers have dedicated mounted LVM filesystems at /var/lib/longhorn

- [ ] iscsid running and iscsi_tcp loaded on each worker

- [ ] Longhorn 1.10.2 deployed; all components healthy

- [ ] Only worker nodes configured for replica storage

- [ ] StorageClass has two replicas and Retain policy

- [ ] 5 GiB PVC and test pod succeed; data survives rescheduling

- [ ] Two healthy replicas verified on separate workers

- [ ] Recovery test documented; backups and restore process defined

## References

Longhorn v1.10 documentation: https://longhorn.io/docs/1.10.0/
Longhorn Helm chart: https://github.com/longhorn/longhorn/tree/v1.10.2/chart
Longhorn project: https://longhorn.io/
