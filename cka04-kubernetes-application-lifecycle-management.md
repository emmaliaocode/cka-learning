# CKA04- Kubernetes Application Lifecycle Management

## 1. Rolling Updates and Rollbacks
#### Deployment 是透過 ReplicaSet 來 Update
```
kubectl get rs
```
```
NAME                  DESIRED   CURRENT   READY   AGE
frontend-86c68cc76c   0         0         0       8m41s
frontend-b9c6c8d8     4         4         4       12m
```
#### Update image
```
kubectl set image deploy [deploName] [containerName]=[newImage]
```
### 1.1. Upgrade Strategy
#### 1.1.1. Recreate
一次更新全部的 Deployment，可能造成 Application 在更新時無法使用。
```yaml
...
  strategy:
    type: Recreate
...
```
```
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  3m48s  deployment-controller  Scaled up replica set frontend-b9c6c8d8 to 4
  Normal  ScalingReplicaSet  51s    deployment-controller  Scaled down replica set frontend-b9c6c8d8 to 0
  Normal  ScalingReplicaSet  17s    deployment-controller  Scaled up replica set frontend-86c68cc76c to 4
```
#### 1.1.2. Rolling Update
一個一個更新 Deployment，可避免 Application 在更新時無法使用。
```yaml
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
...
```
```
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  8m44s  deployment-controller  Scaled up replica set frontend-7494d64f7 to 4
  Normal  ScalingReplicaSet  92s    deployment-controller  Scaled up replica set frontend-b9c6c8d8 to 1
  Normal  ScalingReplicaSet  91s    deployment-controller  Scaled down replica set frontend-7494d64f7 to 3
  Normal  ScalingReplicaSet  91s    deployment-controller  Scaled up replica set frontend-b9c6c8d8 to 2
  Normal  ScalingReplicaSet  69s    deployment-controller  Scaled down replica set frontend-7494d64f7 to 1
  Normal  ScalingReplicaSet  69s    deployment-controller  Scaled up replica set frontend-b9c6c8d8 to 4
  Normal  ScalingReplicaSet  47s    deployment-controller  Scaled down replica set frontend-7494d64f7 to 0
```

### 1.2. Rollout
#### 檢視 Rollout 的狀態
```
kubectl rollout status deploy [deployName]
```
#### 檢視 Rollout 歷史紀錄
```
kubectl rollout history deploy [deployName]
```
```
deployment.apps/frontend 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
### 1.3. Rollback
#### 退回前一個版本
```
kubectl rollout undo deploy [deploy name]
```

## 2. Commands (Docker)
### 2.1. `CMD`
`ubuntu-sleeper.dockerfile`
```dockerfile
FROM Ubuntu
CMD sleep 5
```
Container 創建起來時，`CMD` 會覆蓋原本的 Ubuntu dockerfile 裡的 `CMD`。
```
docker run ubuntu-sleeper
```
### 2.2. `ENTRYPOINT`
將 `ubuntu-sleeper.dockerfile` 的最後一行 `CMD` 改成 `ENTRYPOINT`。
```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
```
Container 創建起來時，會使用傳入的參數來執行指令。
```
docker run ubuntu-sleeper 10
```
### 2.3. `CMD` + `ENTRYPOINT`
如果在執行 `docker run` 時沒有指定參數，就會有 `sleep` 就會報 Error。因此可以混用 `CMD` 及 `ENTRYPOINT` 來指定**預設值**。
```dockerfile
FROM Ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
```
```
docker run ubuntu-sleeper 10
```
#### 指定 Entrypoint
```
docker run --entripoint sleep2.0 ubuntu-sleeper 10
```

## 3. Commands and Arguments (Kubernetes)
### 3.1. `args`
原本在 `docker run` 後面的參數，對應到 Kubernetes 是 `spec.containers[0].args`。
```
docker run ubuntu-sleep-pod 10
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    args: ["10"]
```
```
kubectl run [nginx] --image=[nginx] -- [arg1] [arg2]
```

### 3.2. `command`
原本在 `docker run` 後面的 `--entrypoint`，對應到 Kubernetes 是 `spec.containers[0].command`。
```
docker run --entrypoint sleep2.0 ubuntu-sleep-pod 10
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper
    image: ubuntu-sleeper
    command: ["sleep2.0"]
    args: ["10"]
```
```
kubectl run [nginx] --image=[nginx] --command -- [cmd1] [arg1]
```

## 4. Configure ConfigMap in Applications
設定 Container Environment Variables 基本方法 - `env`。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    env:
    - name: APP_COLOR
      value: pink
    - name: APP_MODE
      value: prod
```
### 4.2. 新增 ConfigMap
#### Imperative 新增
```
kubectl create cm [configmapName] \
--from-literal=key1=value1 \
--from-literal=key2=value2
```
- `--from-literal`: 指定 Key/Value Pairs
```
kubectl create cm [configmapName] \
--from-file=[file]
```

#### Declarative 新增
`config-map.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  key1: value1
  key2: value2
```
```
kubectl create -f [config-map.yaml]
```
### 4.3. 檢視 ConfigMap
```
kubectl get cm
```
```
kubectl describe cm [configmapName]
```
### 4.4. 使用 ConfigMap
#### 使用整個 ConfigMap
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    envFrom:
    - configMapRef:
        name: app-config
```
#### 使用 ConfigMap 裡的部分值
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    env:
    - name: KEY1
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: key1
```

## 5. Configure Secrets in Applications
用來儲存敏感資料，如密碼、金鑰。
- Secrets 並沒有針對內容進行加密 (Encrypted)，只有編碼 (Encoded)。
- 加密 etcd 儲存 Secrets: https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/
- 在相同 Namespace 下，任何可以創建 Pods 或 Deployments 的人都可以取得 Secrets。因此應該要使用 Role-based Access Control (RBAC) 來限制存取權限。
- 使用 AWS, Azure, GCP 時，Secrets 不是儲存在 etcd，而是存在額外的 Secret Provider。

### 5.1. 新增 Secrets
#### Imperative 新增
```
kubectl create secret generic [secretName] \
--from-literal=key1=value1 \
--from-literal=key2=value2 \
--from-literal=key3=value3
```
```
kubectl create secret [secretName] \
--from-file=[file]
```

#### Declarative 新增

`secret-data.yaml`
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
data:
  DB_Host: bXlzcWw=
  DB_User: cm9vdA==
  DB_Password: cGFzd3Jk
```
- `data`: 使用 Encoded Format。

    ```
    echo -n 'mysql' | base64
    ```
    ```
    bXlzcWw=
    ```
```
kubectl create -f [secret-data.yaml]
```
### 5.2. 檢視 Secrets
```
kubectl get secrets
```
```
kubectl describe secrets [secretName]
```
#### 輸出 Secret 的 Key-Value
```
kubectl get secrets [secretName] -o yaml
```
- Decode Value

    ```
    echo -n '[bXlzcWw=]' | base64 -d
    ```
    ```
    mysql
    ```
#### 查看儲存在 etcd 裡的 Secret
```
kubectl exec etcd-controlplane -n kube-system -- sh -c ETCDCTL_API=3 etcdctl --cacert /etc/kubernetes/pki/etcd/ca.crt --cert /etc/kubernetes/pki/etcd/server.crt --key /etc/kubernetes/pki/etcd/server.key get / --prefix --keys-only --limit=10
```
```
/registry/secrets/default/my-secret
```
### 5.3. 使用 Secrets
#### 5.3.1. 使用整個 Secret
#### 以 `envFrom` 設定
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    envFrom:
    - secretRef:
        name: app-secret
```
#### 以 `volume` 設定
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
  volumes:
  - name: app-secret-volume
    secret:
      secretName: app-secret
```
- 以 Volume 方式設定 Secret，會存放在下列位置:
    
    ```
    ls /opt/app-secret-volume
    ```
    ```
    cat /opt/app-secret-volume/DB_Password
    ```
#### 5.3.2. 使用 Secret 的部分值
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    env:
    - name: DB_Password
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_Password
```

## 6. Multi-Container PODs
#### Pod 內的多個 Container 共用 Volume
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: elastic-stack
  labels:
    name: app
spec:
  containers:
  - name: app
    image: kodekloud/event-simulator
    volumeMounts:
    - mountPath: /log
      name: log-volume
  - name: sidecar
    image: kodekloud/filebeat-configured
    volumeMounts:
    - mountPath: /var/log/event-simulator/
      name: log-volume
  volumes:
  - name: log-volume
    hostPath:
      path: /var/log/webapp
      type: DirectoryOrCreate
```
## 7. InitContainers
`initContainers` 的 Container 先**依序創建**完成後，才會創建 `containers` 的。如果 `initContainers` 中有任何一個 Container 創建失敗，Kubernetes 不斷會重啟 Pod，直到成功。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup myservice; do echo waiting for myservice; sleep 2; done;']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
```

---
*Written by Wan-Yu Liao; First version: November 28, 2022*

###### tags: `Kubernetes`