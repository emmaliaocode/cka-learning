# CKA02- Kubernetes Scheduling

## 1. Manual Scheduling to a Specific Node
Master Node 若無 Scheduler， Pod Status 將停留在 `Pending`。
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
cat /var/lib/kubelet/config.yaml
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
### 3.3. 查看調度 Pod 的 Scheduler
```
kubectl get event -o wide
```
```
kubectl logs [scheduler pod name] -n kube-system
```

## 4. Note
### Pod 一直呈現 `Pending` 可能的原因 ...
1. 沒有 Scheduler
2. Taint/Tolerations

---
*Written by Wan-Yu Liao; First version: November 24, 2022*

###### tags: `Kubernetes`