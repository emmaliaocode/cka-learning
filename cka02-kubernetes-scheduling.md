# CKA02- Kubernetes Scheduling

[![hackmd-github-sync-badge](https://hackmd.io/bMDSUyJCQBydcW-V3BT9JA/badge)](https://hackmd.io/bMDSUyJCQBydcW-V3BT9JA)


## 1. Manual Scheduling to a Specific Node
Scheduler 沒有參與調度 Pod。
### 1.1. `nodeName`
在指定 Node 新增 Pod。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  nodeName: node01
```
### 1.2. `kind: Binding`
將已存在的 Pod 調度到指定 Node。
```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target:
  apiVersion: v1
  kind: Node
  name: node01
```
將 YAML 內容轉為 JSON 格式，透過 POST Request 把 Pod 調度到指定的 Node 上。
```
curl \
--header "Content-Type:application/json" \
--request POST \
--data {[bindingObjectJSON]} \
http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
```

## 2. Automatically Scheduling to a Prefer Node
Scheduler 調度 Pod。Master Node 若無 Scheduler， Pod Status 將停留在 `Pending`。
### 2.1. Labels and Selectors
#### 列出符合 `Key=Value` 的 Pod。
```
kubectl get pods --selector=[key1=value1],[key2=value2]
```
#### Labels and Selectors in ReplicaSet
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      labels:
        app: App1
        function: Front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```
#### Labels and Selectors in Service
```yaml
apiVersion: v1
kind: Service
metadata:
  selector:
    app: App1
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```
#### Annotation
Record tool details: name, version, build imformation. 
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
  annotation:
    buildversion: 1.34
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      labels:
        app: App1
        function: Front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```
### 2.2. Taints and Tolerations
> 蟲子 -> Pod
> 人類 -> Node

人類 (Node) 被噴上特定防蟲液後，只有對該種防蟲液具免疫的蟲子 (Pod) 能夠停留在上面。
#### 2.2.1. Taint a Node
#### 新增 Taints
```
kubectl taint nodes [name] [key=value:effect]
```
- `effect`: 當 Pod 無法 Tolerate 時的動作，有三種
    - `NoSchedule`: 只有 Taints/Tolerations 匹配的 Pod 才能被調度到 Node 上。
    - `PreferNoSchedule`: **盡量避免**不匹配的 Taints/Tolerations Pod 被調度到 Node 上。
    - `NoExecute`: 不會將 Taints/Tolerations 未匹配的 Pod 調度到 Node 上，且正在運行中的 Pod 如未匹配，也會遭到驅逐。
#### 移除 Taints
```
kubectl taint nodes [name] [key=value:effect]-
```
#### 2.2.2. Tolerate a Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  tolerations:
  - key: "app"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
```
### 2.3. Node Selectors
#### Label a Node
```
kubectl label nodes [name] [key=value]
```
#### `nodeSelector`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  nodeSelector:
    size: Large
```
### 2.4. Node Affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
            - Medium
```
- `nodeAffinity`:
    - `requiredDuringSchedulingIgnoredDuringExecution`: 指定 Pod 時指定到符合條件的 Node 上，但執行中的 Pod 若不符合條件也不驅逐。
    - `preferredDuringSchedulingIgnoredDuringExecution`: **盡量**把 Pod 指定到符合條件的 Node 上，但執行中的 Pod 若不符合條件也不驅逐。
### 2.5. Resource Request and Limit
若 Pod 若使用資源超過 Limit 就會被 `Terminated`。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
    limits:
      memory: "2Gi"
      cpu: 2
```
#### 單位換算
```
1G (Gigabyte) = 1,000,000,000 bytes
1M (Megabyte) = 1,000,000 bytes
1K (Kilobyte) = 1,000 bytes

1Gi (Gibibtye) = 1,073,741,824 bytes
1Mi (Mebibyte) = 1,048,576 bytes
1Ki (Kibibyte) = 1,024 bytes
```

### 2.6. DaemonSet
確保 Pod 在各個 Node 上的狀態都是 Running，並會跟隨 Node 而建立或移除。在應用 Node 的監控 (Monitoring solution, Logs viewer)，如 kube-proxy。
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: elasticsearch
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        app: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
  selector:
    matchLabels:
      app: fluentd-elasticsearch
```
#### 查看 DaemonSets
```
kubectl get ds -A
```

### 2.7. Static Pods
由 Kubelet，而非 Kube-apiserver 所創建、維持並更新的 Pod。Kubeadm 就是透過 Static Pod 來創建 Kubernetes Cluster。

#### 2.7.1. 查看 Static Pod YAML 放置位置
印出 Kubelet 的 Configuration，看 `staticPodPath`，通常 Static Pod YAML 會放在 `/etc/kubernetes/manifests`。
```
ps -aux | grep kubelet | grep --color config
```
```!
root        2336  0.0  0.0 4227428 99728 ?       Ssl  13:11   0:34 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=k8s.gcr.io/pause:3.7
```
```
cat /var/lib/kubelet/config.yaml | grep static
```
```
staticPodPath: /etc/kubernetes/manifests
```
```
ls /etc/kubernetes/manifests
```
```!
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```
#### 2.7.2. Static Pod 命名規則
Static Pods 的名字會以 Node Name 做結尾。例如 `etcd-controlplane`, `kube-apiserver-controlplane`, `kube-controller-manager-controlplane`, `kube-scheduler-controlplane`。
```
kubectl get pod -n kube-system
```
```
NAMESPACE     NAME                                   READY   STATUS    RESTARTS   AGE
kube-system   coredns-6d4b75cb6d-brzzn               1/1     Running   0          22m
kube-system   coredns-6d4b75cb6d-v68p7               1/1     Running   0          22m
kube-system   etcd-controlplane                      1/1     Running   0          22m
kube-system   kube-apiserver-controlplane            1/1     Running   0          22m
kube-system   kube-controller-manager-controlplane   1/1     Running   0          22m
kube-system   kube-flannel-ds-f55cg                  1/1     Running   0          22m
kube-system   kube-flannel-ds-h7qzl                  1/1     Running   0          22m
kube-system   kube-proxy-6bhdx                       1/1     Running   0          22m
kube-system   kube-proxy-h574v                       1/1     Running   0          22m
kube-system   kube-scheduler-controlplane            1/1     Running   0          22m
```
#### 2.7.3. Static Pods vs DaemonSets

| Type        | Created by     | Application                       | Ignored by Kube-Scheduler  |
| ----------- | -------------- | --------------------------------- | --- |
| Static Pods | Kubelet        | Control manager                   | Yes |
| DaemonSets  | Kube-apiserver | Monitering agents, Logging agents | Yes |


## 3. Multiple Schedulers
### 3.1. 以 Pod 部署 Customized Scheduler
`my-custom-scheduler.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  containers:
  - name: kube-scheduler
    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --config=/etc/kubernetes/my-scheduler/my-scheduler-config.yaml
    volumeMounts:
    - name: config-volume
      mountPath: /etc/kubernetes/my-scheduler
  volumes:
  - name: config-volume
    hostPath:
      path: /etc/kubernetes/my-scheduler
      type: directory
```
`my-scheduler-config.yaml`
```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: my-scheduler
leaderElection:
  leaderElect: true
  resourceNamespace: kube-system
  resourceName: lock-object-my-scheduler
```
```
kubectl create -f my-custom-scheduler.yaml
```
### 3.2. 以 Deployment 部署 Customized Scheduler
> [Doc: Configure Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
### 3.3. `schedulerName`
透過特定 Scheduler 來調度 Pod。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - image: nginx
    name: nginx
```
### 3.3. 查看調度 Pod 的 Scheduler
```
kubectl get event -o wide
```
```
LAST SEEN   TYPE      REASON                    OBJECT              SUBOBJECT                SOURCE                                    MESSAGE                                                               FIRST SEEN   COUNT   NAME
26m         Normal    NodeHasSufficientMemory   node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasSufficientMemory              26m          7       controlplane.1733be07ee6cb4f8
26m         Normal    NodeHasNoDiskPressure     node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasNoDiskPressure                26m          7       controlplane.1733be07ee6cf8ba
26m         Normal    NodeHasSufficientPID      node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasSufficientPID                 26m          6       controlplane.1733be07ee6d2e4f
26m         Normal    NodeAllocatableEnforced   node/controlplane                            kubelet, controlplane                     Updated Node Allocatable limit across pods                            26m          1       controlplane.1733be08b3c0bf8c
25m         Normal    Starting                  node/controlplane                            kubelet, controlplane                     Starting kubelet.                                                     25m          1       controlplane.1733be0f5a701779
25m         Warning   InvalidDiskCapacity       node/controlplane                            kubelet, controlplane                     invalid capacity 0 on image filesystem                                25m          1       controlplane.1733be0f5fa25327
25m         Normal    NodeHasSufficientMemory   node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasSufficientMemory              25m          1       controlplane.1733be0f65bffa1e
25m         Normal    NodeHasNoDiskPressure     node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasNoDiskPressure                25m          1       controlplane.1733be0f65c04d4b
25m         Normal    NodeHasSufficientPID      node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeHasSufficientPID                 25m          1       controlplane.1733be0f65c061bb
25m         Normal    NodeAllocatableEnforced   node/controlplane                            kubelet, controlplane                     Updated Node Allocatable limit across pods                            25m          1       controlplane.1733be0fb2857169
25m         Normal    RegisteredNode            node/controlplane                            node-controller                           Node controlplane event: Registered Node controlplane in Controller   25m          1       controlplane.1733be11b3ba02dc
25m         Normal    Starting                  node/controlplane                            kube-proxy, kube-proxy-controlplane                                                                             25m          1       controlplane.1733be13235d22af
25m         Normal    NodeReady                 node/controlplane                            kubelet, controlplane                     Node controlplane status is now: NodeReady                            25m          1       controlplane.1733be1433246dd0
90s         Normal    Scheduled                 pod/nginx                                    my-scheduler, my-scheduler-my-scheduler   Successfully assigned default/nginx to controlplane                   90s          1       nginx.1733bf641f0ea55e
80s         Normal    Pulling                   pod/nginx           spec.containers{nginx}   kubelet, controlplane                     Pulling image "nginx"                                                 80s          1       nginx.1733bf6684026b34
79s         Normal    Pulled                    pod/nginx           spec.containers{nginx}   kubelet, controlplane                     Successfully pulled image "nginx" in 311.46565ms                      79s          1       nginx.1733bf669693738a
78s         Normal    Created                   pod/nginx           spec.containers{nginx}   kubelet, controlplane                     Created container nginx                                               78s          1       nginx.1733bf66e8d245d1
71s         Normal    Started                   pod/nginx           spec.containers{nginx}   kubelet, controlplane                     Started container nginx                                               71s          1       nginx.1733bf68800f550d
```
```
kubectl logs [schedulerPodName] -n kube-system
```


## 4. Note
### Pod 一直呈現 `Pending` 可能的原因 ...
1. 沒有 Scheduler
2. Taint/Tolerations

---
*Written by Wan-Yu Liao; First version: November 24, 2022*

###### tags: `Kubernetes`