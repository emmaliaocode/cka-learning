# CKA03- Kubernetes Logging and Monitoring

[![hackmd-github-sync-badge](https://hackmd.io/XaRTvGiuRtOfMd0q5_-DWg/badge)](https://hackmd.io/XaRTvGiuRtOfMd0q5_-DWg)


## 1. Monitor Cluster Components
### 1.1. Metrics Server
Kubelet 裡的 **cAdvisor (Container Advisor)** 會負責擷取 Pod 的 Performance Metrics 資料，並透過 Kubelet API 傳給 Metrics Server。

Metrics Server 是 In-memory 的所以看不到歷史紀錄。需要透過第三方軟體來紀錄，常見軟體如下: 
- Open-source: Metrics Server, Prometheus, Elastic Stack
- Proprietary solutions: Datadog, Dynatrace
### 1.2. 部署 Metrics Server
```
git clone https://github.com/kubernetes-incubator/metrics-server.git
```
```
kubectl create –f deploy/1.8+/
```
創建一系列的 Pods, Services, Roles 以建立 Metrics Server，拉取 Cluster 內所有 Node 的 Performance Metrics。
### 1.3. 查看 Performance Metrics
列出所有 Node 或 Pod 的 CPU, Memroy 使用量資訊。
```
kubectl top node
```
```
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
controlplane   295m         0%     1179Mi          0%        
node01         24m          0%     274Mi           0%
```
```
kubectl top pod
```
```
NAME       CPU(cores)   MEMORY(bytes)   
elephant   19m          32Mi            
lion       1m           18Mi            
rabbit     122m         252Mi
```

## 2. Logs
如果一個 Pod 內有多個 Container 需要在 `kubectl logs [podName]` 後再指定 Container Name。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
  - name: image-processor
    image: some-image-processor
```
```
kubectl logs (-f) [podName] ([containerName])
```


---
*Written by Wan-Yu Liao; First version: November 28, 2022*

###### tags: `Kubernetes`