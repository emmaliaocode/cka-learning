# CKA07- Kubernetes Storage

[![hackmd-github-sync-badge](https://hackmd.io/5JpvIgI7SIqvIx11zD4jGQ/badge)](https://hackmd.io/5JpvIgI7SIqvIx11zD4jGQ)

## 1. Storage in Docker
Image Layer vs Container Layer
- Image Layer -> Read-only
- Container Layer -> Read, Write

### 1.1. Storage Drivers
常見的 Storage Driver 種類: AUFS, ZFS, BTRFS, Device Mapper, Overlay, Overlay2。Docker 會依據 OS 自動選擇適合的 Driver (Ubuntu 預設用 AUFS)。
#### 1.1.1. Volumn Mount
在 `/var/lib/docker/volumes` 底下新增一個 Volume (Volume Driver 負責) 並 Mount 到 Container。
```
docker volume create data_volume
```
```
docker run -v data_volume:/var/lib/mysql mysql
```
#### 1.1.2. Bind Mount
將既有的資料夾 Mount 到 Container。
```
docker run -v /data/mysql:/var/lib/mysql mysql
```
或用 `--mount`:
```
docker run \
--mount type=bind,source=/data/mysql,target=/var/lib/mysql mysql
```
### 1.2. Volume Drivers
於 Local 或第三方 Storage Solution (如 GlusterFS, RexRay, Convoy, Azure File Storage 等等) 上建立 Volume。

部分 Volume Driver 支援不同的 Storage Provider，如 RexRay 支援 AWS EBS, S3, Google Persistent Disk。
```
docker run -it \
--name mysql \
--volume-driver rexray/ebs
--mount src=ebs-vol,target=/var/lib/mysql \
mysql
```
## 2. Container Storage Interface (CSI)
- Container Runtime Interface (CRI): Orchestration Tools (如 Kubernetes) 與 Container Runtime (如 Docker, rkt, cri-o) 之間溝通的 Standard。
- Container Network Interface (CNI)
- **Container Storage Interface (CSI)**: 根據 CSI Standard 開發出各種 Storage Driver。

#### CSI Standard
1. Remote Procedure Call (RPC)
2. CreateVolume & DeleteVolume
3. ControllerPublishVolume

一個有 Volume 需求的 Pod 被建立後，Kubernetes 會 Call RPC，把建立 Volume 的細節傳給 Storage Driver 後，由 Storage Driver 建立 Volume 並回傳結果。

## 3. Volumes
#### `hostPath`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh","-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    hostPath:
      path: /data
      type: directory
```
- `volumes`: 支援各種 Storage Solution，如 NFS, GlusterFS, Flocker, Ceph FS, AWS EBS, Azure Desk, Google Persistent Disk 等等。

#### `awsElasticBlockStore`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
spec:
  containers:
  - image: alpine
    name: alpine
    command: ["/bin/sh","-c"]
    args: ["shuf -i 0-100 -n 1 >> /opt/number.out;"]
    volumeMounts:
    - mountPath: /opt
      name: data-volume
  volumes:
  - name: data-volume
    awsElasticBlockStore:
      volumeID: [volume-id]
      fsType: ext4
```

## 4. Persistent Volumes (PV)
`pv-definition.yaml`
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-voll
spec:
  persistentVolumeReclaimPolicy: Retain
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 1Gi
  awsElasticBlockStore:
    volumeID: [volume-id]
    fsType: ext4
```
- `persistentVolumeReclaimPolicy`
    - `Retain`: 預設值。PV 在 PVC 被刪除後仍會保留，其他的 Claim 不能 Reuse 這個 PV。
    - `Delete`: PV 在 PVC 被刪除後也會一並被刪除。
    - `Recycle`: PV 在 PVC 被刪除後，與其他 PVC 綁定之前會先把 PV 上的資料都刪除。
```
kubectl create -f pv-definition.yaml
```
```
kubectl get persistentvolume
```

## 5. Persistent Volume Claims (PVC)
PV and PVC
- Persistent Volume (PV) -> Admin 建立
- Persistent Volume Claim (PVC) -> User 建立來使用 PV

### 5.1. Binding PV and PVC
PVC 建立後，Kubernetes 根據 PVC 定義的需求來綁定 PV 與 PVC:
- Sufficient Capacity
- Access Modes: `ReadWriteMany`/`ReadWriteOnce`
- Volume Modes
- Storage Class
- Selector

一個 PVC 只會綁定一個 PV。若無 PV 符合 PVC 的需求，PVC 會一直呈現 `Pending`。

`pvc-definition.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```
```
kubectl get persistentvolumeclaim
```
```
kubectl delete persistentvolumeclaim [myclaim]
```
- PVC 被刪除之後，PV 是否被刪除是根據 PV 裡定義的 `persistentVolumeReclaimPolicy`。

### 5.2. Using PVCs in Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myfrontend
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: mypd
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
```

## 6. Storage Class
Automatically provision storage.
- Dynamic provisioning of volumes: Storage Class 會自動建立 PV (不用寫 PV definition file)，PV 建立後會與 PVC 綁定，Pod 便可以使用。

`sc-definition.yaml`
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: none
```
`pvc-definition.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: google-storage
  resources:
    requests:
      storage: 500Mi
```

---
*Written by Wan-Yu Liao; First version: December 10, 2022*

###### tags: `Kubernetes`