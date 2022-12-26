# CKA05- Kubernetes Cluster Maintenance

[![hackmd-github-sync-badge](https://hackmd.io/XGx9fgFjSkqc1JYCc7xNPQ/badge)](https://hackmd.io/XGx9fgFjSkqc1JYCc7xNPQ)

## 1. OS Upgrade
Controller Manager 設定 `pod-eviction-timeout=5m`，因此如果 Node 超過五分鐘沒有復原，就會把該 Node 上的 Pod 移到其他的 Node 上。
### 1.1. Drain
Terminate Node 上的 Pods，並在其他 Node 上重新 Create。Node 會被標示為 Unschedulable，表示沒有任何 Pod 可以被放到這個 Node 上。
```
kubectl drain [node name]
```
:::info
**Drain Node 可能遇到的 Error**
#### 1. 用 DaemonSet 部署的 Pod 無法被刪除:
```
node/node01 cordoned
error: unable to drain node "node01" due to error:cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/kube-flannel-ds-nwrtp, kube-system/kube-proxy-dgd7j, continuing command...
There are pending nodes to be drained:
 node01
cannot delete DaemonSet-managed Pods (use --ignore-daemonsets to ignore): kube-system/kube-flannel-ds-nwrtp, kube-system/kube-proxy-dgd7j
```
解法: 加上 `--ignore-daemonsets`，在轉移 Pod 時忽略用 DaemonSet 部署的 Pod。

#### 2. Pod 沒被任何一種 Controller 控制，刪掉就不見了:
```
node/node01 cordoned
error: unable to drain node "node01" due to error:cannot delete Pods declare no controller (use --force to override): default/hr-app, continuing command...
There are pending nodes to be drained:
 node01
cannot delete Pods declare no controller (use --force to override): default/hr-app
```
解法: 將 Pod 改為透過 Deployment 或 ReplicaSet 來部署。
:::

### 1.2. Cordon
不會 Terminate Node 上的 Pods，只會把 Node 標示成 Unschedulable。
```
kubectl cordon [node name]
```
### 1.3. Uncordon
使後面創建的 Pod 能再被 Schedule 到這個 Node 上。
```
kubectl uncordon [node name]
```

## 2. Kubernetes Software Versions
```
  v1  .  11  .  3
Major  Minor   Patch
       | Features & functionalities (Every few months)
               | Bug fixes
```
從 Kubernetes 官方下載並解壓縮後包含**相同版本**的以下 Components:
- Kube-apiserver
- Controller-manager
- Kube-scheduler
- Kubelet
- Kube-proxy

其他的 Components 版本不一定會跟上面一樣:
- Kubectl
- etcd (不屬於 Kubernetes Project)
- CoreDNS (不屬於 Kubernetes Project)

## 3. Cluster Upgrade Process
### 3.1. Version Limitations
- Kube-apiserver: `X` --> v1.10
- Controller-manager: `X-1` --> v1.9 or v1.10
- Kube-scheduler: `X-1` --> v1.9 or v1.10
- Kubelet: `X-2` --> v1.9 or v1.10
- Kube-proxy: `X-2` --> v1.8 or v1.9 or v1.10

### 3.2. Upgrade Strategies
- 一次更新所有 Node: 服務可能會中斷。
- 批次更新部分 Node: 暫時把 Pods 放到其他 Node 上，更新完後再放回來。
- 新增裝有新版 Kubernetes 的 Node，把 Pod 放到新版本的 Node 後再移除舊的 Node (在 Cloud 上的開發適合用這種方式)。

## 4. Upgrade Kubeadm Cluster
### 4.1. 查看版本
#### 4.1.1. 查看目前版本
```
kubeadm version
```
```!
kubeadm version: &version.Info{Major:"1", Minor:"24", GitVersion:"v1.24.0", GitCommit:"4ce5a8954017644c5420bae81d72b09b735c21f0", GitTreeState:"clean", BuildDate:"2022-05-03T13:44:24Z", GoVersion:"go1.18.1", Compiler:"gc", Platform:"linux/amd64"}
```
Kubeadm 的版本是跟著 Kubernetes。`GitVersion:"v1.24.0"` 表示 Kubernetes 是 `v1.24.0` 版。

#### 4.1.2. 查看可更新的版本
```
apt-cache madison kubeadm  # 看有哪些版本可以用
```
#### 4.1.3. 查看可更新的穩定版本
```
kubeadm upgrade plan
```
- `"remote version"`
### 4.2. 更新 Master Node
#### 4.2.1. 升級 Master Node 版本
```
kubectl drain controlplane
```
```
apt-mark unhold kubeadm
apt update
apt install -y kubeadm=1.25.0-00
apt-mark hold kubeadm
kubeadm upgrade apply v1.25.0
```
```
kubectl get nodes
```
更新完後，會看到該 Node 的 `VERSION` 還是舊的，是因為**顯示的是 Kubelet 的版本**。

#### 4.2.1. 升級 Master Node 的 Kubelet
接下來手動更新 Kubelet。(Master Node 有可能沒有 Kubelet)
```
apt-mark unhold kubelet kubectl
apt update
apt install -y kubelet=1.25.0-00 kubectl=1.25.0-00
apt-mark hold kubelet kubectl
```
```
systemctl daemon-reload  # 重新加載服務
systemctl restart kubelet  # 重啟 kubelet
```
#### 4.2.3. Uncordon Node
```
kubectl uncordon controlplane
```
```
kubectl get nodes
```
### 4.3. 更新 Worker Node
#### 4.3.1. 升級 Worker Node 版本
```
kubectl drain node01 --ignore daemonsets  # 一定要在 Master Node 執行 !
```
```
ssh [user]@[node ip]
```
```
apt-mark unhold kubeadm
apt update
apt install -y kubeadm=1.25.0-00
apt-mark hold kubeadm
kubeadm upgrade node
```
```
kubectl get nodes
```
#### 4.3.2. 升級 Worker Node 的 Kubelet
```
apt-mark unhold kubelet
apt update
apt install -y kubelet=1.25.0-00
apt-mark hold kubelet
```
```
systemctl daemon-reload
systemctl restart kubelet
```
#### 4.3.3. Uncordon Node
```
kubectl uncordon node01  # 一定要在 Master Node 執行 !
```
```
kubectl get nodes
```

## 5. Backup and Restore
### 5.1. Resource Configs
Objects 有可能是透過 Imperative 或 Declarative 建立，目前 Cluster 的 Objects 未必與 Definition File 裡的內容吻合，因此在 Backup 的時候不應該只儲存 Definition File。
#### 5.1.1. 備份 YAML
匯出目前 Cluster 上的所有 Pods, Deployments, Services 的 YAML。
```
kubectl get all -A -o yaml > all-deploy-services.yaml
```
#### 5.1.2. 備份工具
- Velero (前身是 HeptIO 的 ARK)
### 5.2. etcd
etcd Topology 的種類:
1. **Stacked ETCD Topology**
    - `kube-system` Namespace **有** `etcd` Pod。
2. **External ETCD Topology**
    - `kube-system` Namespace **沒有** `etcd` Pod。
    - SSH 到 Node，`/etc/kubernetes/manifests` 沒有 etcd 的 Static pod configuration。
    - `kube-apiserver` 的 `--etcd-server` 為外部 Server。
> etcd Toplogy 差異: https://ithelp.ithome.com.tw/articles/10217811

#### 5.2.1. 查看 etcd 存放位置
#### For stacked etcd
```
kubectl describe pod -n kube-system [etcdPodName]
```
```yaml
Command:
  ...
  --data-dir=/var/lib/etcd
  ...
```
#### For external etcd
```
ssh [etcdServerIp]
ps aux | grep etcd
```
```
/usr/local/bin/etcd --name etcd-server --data-dir=/var/lib/etcd-data ...
```
#### 5.2.2. 建立 etcd Snapshot
```
ETCDCTL_API=3 etcdctl \
--endpoints=[https://127.0.0.1:2379] \
--cacert=[caCert] \
--cert=[cert] \
--key=[key] \
snapshot save [/path/to/snapshot.db]
```
- `cacert`, `cert`, `key` 這三把鑰匙通常都會放在: `/etc/kubernetes/pki`。
    - `--cacert`: verify certificates of TLS-enabled secure servers using this CA bundle
    - `--cert`: identify secure client using this TLS certificate file
    - `--key`: identify secure client using this TLS key file
#### 5.2.3. etcd Snapshot 狀態
```
ETCDCTL_API=3 etcdctl \
--endpoints=[https://127.0.0.1:2379] \
--cacert=[caCert] \
--cert=[cert] \
--key=[key] \
snapshot status [/path/to/snapshot.db]
```

> ETCD FAQ: https://github.com/kodekloudhub/community-faq/blob/main/docs/etcd-faq.md

#### 5.2.4. 復原 etcd
#### For stacked etcd
在 Master Node 下指令:
```
ETCDCTL_API=3 etcdctl \
--endpoints=[https://127.0.0.1:2379] \
--cacert=[ca cert] \
--cert=[cert] \
--key=[key] \
snapshot restore [snapshotFile] --data-dir [newEtcdDir]
```
復原 etcd 時，會新增新的 Cluster Configuration，避免新舊混在一起。因此要修改 etcd 的 Definition File，把 `--data-dir` 修改成新的路徑 (`/var/lib/etcd-from-backup`)。
```
vi /etc/kubernetes/manifests/etcd.yaml
```
```yaml
...
  volumes:
  - hostPath:
    name: etcd-data
    path: /var/lib/etcd-from-backup
...
```
改完後，Controller Manager 跟 Scheduler 都會重啟。
```
watch "crictl ps | grep etcd"
```
如果 etcd Pod 沒有變為 `Ready 1/1`，就把原本的 Pod 刪掉重開。
```
kubectl delete pod -n kube-system etcd-controlplane
```
#### For external etcd
在 Database Server 執行上面的 Restore 指令。
```
ETCDCTL_API=3 etcdctl \
--endpoints=[https://127.0.0.1:2379] \
--cacert=[caCert] \
--cert=[cert] \
--key=[key] \
snapshot restore [/opt/cluster2.db] --data-dir=[/var/lib/etcd-data-new]
```
```
{"level":"info","ts":1669863818.3708405,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/opt/cluster2.db","wal-dir":"/var/lib/etcd-data-new/member/wal","data-dir":"/var/lib/etcd-data-new","snap-dir":"/var/lib/etcd-data-new/member/snap"}
{"level":"info","ts":1669863818.391143,"caller":"mvcc/kvstore.go:388","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":2854}
{"level":"info","ts":1669863818.3990204,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"cdf818194e3a8c32","local-member-id":"0","added-peer-id":"8e9e05c52164694d","added-peer-peer-urls":["http://localhost:2380"]}
{"level":"info","ts":1669863818.415179,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/opt/cluster2.db","wal-dir":"/var/lib/etcd-data-new/member/wal","data-dir":"/var/lib/etcd-data-new","snap-dir":"/var/lib/etcd-data-new/member/snap"}
```
把 `--data-dir` 修改成新的路徑。
```
vi /etc/systemd/system/etcd.service
```
修改資料夾權限為 `etcd:etcd`。
```
chown -R etcd:etcd /var/lib/etcd-data-new
```
重啟 etcd 服務。
```
systemctl daemon-reload
systemctl restart etcd
```

## Note
### 查看 Node 上有多少 Cluster
```
kubectl config get-clusters
```
```
kubectl config view
```
### 查看目前顯示的 Context 是哪個 Cluster
```
kubectl config current-context
```
### 查看這個 Cluster 部署在哪些 Node 上
```
kubectl get nodes
```
#### 查看這個 etcd Server 的 Cluster
```
ETCDCTL_API=3 etcdctl \
--endpoints=[https://127.0.0.1:2379] \
--cacert=[caCert] \
--cert=[cert] \
--key=[key] \
member list
```


---
*Written by Wan-Yu Liao; First version: November 29, 2022*

###### tags: `Kubernetes`