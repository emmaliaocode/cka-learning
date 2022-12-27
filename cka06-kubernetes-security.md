# CKA06- Kubernetes Security

[![hackmd-github-sync-badge](https://hackmd.io/oxhV2yBQTr6kxpc74H2Ufg/badge)](https://hackmd.io/oxhV2yBQTr6kxpc74H2Ufg)

## 1. Kubernetes Security Primitives
Kubernetes 安全管理:
1. **Authentication (驗證): Who can access?**: 使用者是否是其所宣稱的那個人。
	- Username & Passwords (Files)
	- Username & Tokens (Files)
	- TLS Certificates
	- External Authentication providers - LDAP
	- Service Accounts
2. **Authorization (授權): What can they do?**: 根據使用者的角色授予應有的權限。
	- Role-based Access Control (RBAC) Authorization
	- Attribute-based access control (ABAC) Authorization
	- Node Authorization
	- Webhook Mode

- Cluster 內部的 Pod 預設是可以跟其他 Pod 互相溝通，但也可以設定 **Network Policy** 來限制，讓 Pod 間無法互相溝通。

## 2. Authentication
可以分成: 
1. **User Accounts**: Admins, Developers
2. **Service Accounts**: Thirty-party app for integration (CKAD 範圍，這邊不討論)

### 2.1. User Accounts (Deprecated in 1.19)
不論是用 `kubectl` 或是 `curl`，Kube-apiserver 會在處理需求前先驗證使用者身份。
#### 2.1.1. Static Password File
把使用者名稱、密碼、ID 放在 CSV 裡，在 Kube-apiserver 的設定加上選項 `--basic-auth-file` 貼上 CSV 的路徑。

`user-password.csv`
```
password123,user1,u0001
password123,user2,u0002
password123,user3,u0003
```
設定 User Authentication 後用 `curl` 打 Kube-apiserver 就要在後面帶 `-u` 選項。
```
curl -v -k \
https://[masterNodeIP]:6443/api/v1/pods \
-u "[userName]:[password]"
```
#### 2.1.2. Static Token File (Deprecated in 1.19)
在 Kube-apiserver 的設定加上選項 `--token-auth-file` 貼上 CSV 的路徑。

`user-token.csv`
```
jiwojw3io2fj92ojfioefj,user1,u0001
jfo230ufjklfklsnfkldss,user2,u0002
nkjrhwljrpqirdjskgjiow,user3,u0003
```
`curl` 打 Kube-apiserver 要在後面帶 `--header` 選項。
```
curl -v -k \
https://[masterNodeIP]:6443/api/v1/pods \
--header "Authrization: Bearer [token]"
```
#### 2.1.3. Certificates
以下詳細說明。

## 3. TLS Certificates
保持網際網路連線安全以及防止在兩個系統之間發送的所有敏感資料被罪犯讀取及修改任何傳輸的資訊，包括潛在的個人詳細資料。兩個系統可以是伺服器與用戶端 (例如購物網站與瀏覽器)，或者伺服器至伺服器 (例如，含有個人身份資訊或含有薪資資訊的應用程式)。(https://www.websecurity.digicert.com/zh/hk/security-topics/what-is-ssl-tls-https)
### 3.1. Encryption Type
#### 3.1.1. Symmetric Encryption
用同一把鑰匙 Encrypt 跟 Decrypt 資料。因為傳送方會把鑰匙一併傳給接收方，駭客在過程中可能攔截鑰匙就能解密其加密的資訊。

#### 3.1.2. Asymmetric Encryption
產生兩把鑰匙，Public Key (想像成是 Lock) 跟 Private Key。**只有對應的 Private Key 才能解密用 Public Key 加密的資訊。**
- Public Key (`*.crt`, `*.pem`): 可公開
- Private Key (`*.key`, `*-key.pem`): 不可公開

### 3.2. TLS 應用 - SSH & Web Server
#### 3.2.1. SSH: Asymmetric Encryption
裝置產生 Public & Private Key。
```
ssh-keygen
```
在 Server 的 `~/.ssh/authorized_key` 加入 Public Key (Lock)。
```
cat id_rsa.pub >> ~/.ssh/authorized_key
```
一台裝置的 Public Key 可以放在多台 Server 上，讓那台裝置能 SSH 到不同的 Server；一台 Server 也可以存有不同裝置的 Public Key，讓不同的裝置都能 SSH 到那一台 Server。

#### 3.2.2. Web Server: Asymmetric Encryption + Symmetric Encription
Server & Client 先產生各自的 Public & Private Key
```
openssl genrsa -out my-key.key 1024
openssl genrsa -in my-key.key -pubout my-key.pem
```
接下來 Client 第一次透過 HTTPS 訪問 Server 時，會取得 Server 的 Public Key。

最後，Client Browser 利用 Server Public Key 加密 Client Symmetric Key 並傳給 Server。Server 接收到之後用 Server Private Key 解密，得到 Client Symmetric Key，後續就能用 Client Symmetric Key 做加密跟解密。

### 3.3. Certificate Authority
Certificate Authority (CA) 組織頒發 Certificate，目的是確認你就是該 Domain 的擁有者，藉此確保拜訪的網站不是駭客搭建的惡意網站。

Browser 會驗證 Certificate Server 傳送 Public Key 時附帶的 Certificate。如果:
1. 沒有附帶 Certificate 或
2. Certificate 不是經由 CA 組織 (Symantec, DigiCert, Comodo, GlobalSign 等) 頒發

就會被認為是不安全的。

**參與取得驗證的 CA, Server, 流程等，統稱為 Public Key Infrastructure (PKI)**。

:::info
**Browser 如何驗證 CA ?**
1. 各 CA 組織會產生各自的 Public & Private Key。
2. 各 CA 用 Private Key 頒發 Certificate，CA Public Key 內建在 Browser。
3. Browser 使用 Public Key 來確認 Certificate 是合法 CA 所頒發。
:::

:::info
**Server 如何驗證 Client ?**
1. Client 建立 Key。
2. Key 經由 CA 驗證後傳到 Server。
:::

```
* Issued To
Common Name (CN)	udemy.com
Organization (O)	Cloudflare, Inc.
Organizational Unit (OU)	<Not Part Of Certificate>

* Issued By
Common Name (CN)	Cloudflare Inc ECC CA-3
Organization (O)	Cloudflare, Inc.
Organizational Unit (OU)	<Not Part Of Certificate>

* Validity Period
Issued On	Tuesday, June 28, 2022 at 8:00:00 AM
Expires On	Thursday, June 29, 2023 at 7:59:59 AM

* Fingerprints
SHA-256 Fingerprint	27 AC 79 24 55 2B 9D 8C 12 B5 AF 60 79 FF 36 B3
10 4F FF DA 68 7B 2D 8E 0D A4 6F EF 8E 86 A1 69
SHA-1 Fingerprint	50 01 E0 B3 DB 1D AD 72 68 BA 26 27 94 1A 0A 59
66 D9 C1 EF
```

## 4. TLS in Kubernetes
Kubernetes 的 TLS Certificates 可以分成三種:
1. **Server Certificates** - Server 建立的 Public & Private Key
2. **Client Certificates** - Client 建立的 Public & Private Key
3. **Root Certificates** - CA 建立的 Public & Private Key

![](https://i.imgur.com/Agseuo3.jpg)

![](https://i.imgur.com/VTXkNZs.jpg)

## 5. TLS in Kubernetes - Certificate Creation
TLS Certificates 建立步驟:
1. Generate Keys
2. Certificate Signing Request (CSR)
3. Sign Certificates
### 5.1. Client Certificate
#### 5.1.1. CA
Certificate name: `/CN=KUBERNETES_CA`
```
# Generate Keys
openssl genrsa -out [ca.key] 2048

# Certificate Signing Request
openssl req -new -key [ca.key] -subj "[/CN=KUBERNETES_CA]" -out [ca.csr]

# Sign Certificates
openssl x509 -req -in [ca.csr] -signkey [ca.key] -out [ca.crt]
```
#### 5.1.2. Admin User
Certificate name: `/CN=kube_admin`
Group: `/O=system:masters`
```
# Generate Keys
openssl genrsa -out [admin.key] 2048

# Certificate Signing 
openssl req -new -key [admin.key] -subj "[/CN=kube_admin/O=system:masters]" -out [admin.csr]

# Sign Certificates
openssl x509 -req -in [admin.csr] -CA [ca.crt] -CAkey [ca.key] -out [admin.crt]
```
- `-subj`
    - `CN`: Common name
    - `O`: Group name
- `-CA`, `-CAkey`: 用 Kubernetes CA 讓 Athentication 在這個 Cluster 是有效的。

Scheduler, Controller, Kube-proxy 比照辦理，前兩個都是 System Component，因此命名 `/CN=SYSTEM:KUBE-SCHEDULER` 以及 `/CN=SYSTEM:KUBE-CONTROLLER-MANAGER`；Kube-proxy 則命名為 `/CN=KUBE-PROXY`。
#### 5.1.3. Kubelet-client
Certificate name: `/CN=system:node:[nodeName]`
Group: `/O=system:nodes`
```
# Generate Keys
openssl genrsa -out [system:node:[nodeName].key] 2048

# Certificate Signing 
openssl req -new -key [system:node:[nodeName].key] -subj "/CN=[nodeName]/O=system:node" -out [system:node:[nodeName].csr]

# Sign Certificates
openssl x509 -req -in [system:node:[nodeName].csr] -CA [ca.crt] -CAkey [ca.key] -out [system:node:[nodeName].crt]
```
### 5.2. Server Certificate
#### 5.2.1. Kube-apiserver
Certificate name: `/CN=kube-apiserver`
```
# Generate Keys
openssl genrsa -out [apiserver.key] 2048

# Certificate Signing 
openssl req -new -key [apiserver.key] -subj "[/CN=kube-apiserver]" -out [apiserver.csr] -config [openssl.cnf]

# Sign Certificates
openssl x509 -req -in [apiserver.csr] -CA [ca.crt] -CAkey [ca.key] -out [apiserevr.crt]
```
- `-config`: Openssl config file

`openssl.cnf`
```y
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation,
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```
#### 5.2.2. etcd
Certificate name: `/CN=ETCD-SERVER` & `/CN=ETCD-PEER`
在 High-availability 環境下，etcd Cluster 要能互相溝通。因此在相同 etcd Cluster 下的 etcd Member 就要產生額外的 `etcdpeer.crt`。
#### 5.2.3. Kubelet-server
Certificate name: `/CN=[nodeName]`
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
authentication:
  x509:
    clientCaFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "$(POD_CIDR)"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
```
### 5.3. 使用 Certificate
有 `ca.crt`、`*.key` 以及 `*.crt` 後，`curl` 不需要帶帳號密碼。
```
curl https://kube-apiserver:6443/api/v1/pods \
--cacert [ca.crt] \
--key [admin.key] \
--cert [admin.crt]
```
也可以用 `Config` YAML 定義。
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ca.crt
    server: https://kube-apiserver:6443
  name: kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

## 6. View Certificate Detail
依據 Cluster 建立方式的不同，到不同的地方去找 Configuration。
- **From scratch**: `cat /etc/systemd/system/kube-apiserver.service`
- **Kubeadm**: `cat /etc/kubernetes/manifests/kube-apiserver.yaml`

### 6.1. View Certificate
```
openssl x509 -in [/etc/kubernetes/pki/apiserver.crt] -text -noout
```
### 6.2. View Log
#### Kubeadm
```
kubectl logs etcd-master
```
#### Container Runtime Interface (CRI)
```
crictl ps -a
```
```
crictl logs [containerId]
```
#### From scratch
```
journalctl -u [etcd.service] -1
```
#### Docker
```
docker ps -a
```
```
docker log [containerId]
```

## 7. Certificate Signing Request (CSR)
Kubernetes Certificates API 透過 `CertificateSigningRequest` Object 簡化新增 Certificates 的流程。

儲放 Certificate Key 的 CA Server 位於 Master Node ，可以透過 API 經由 Controller-Manager 的 `CSR-APPROVING` Controller 跟 `CSR-SIGNING` Controller 來執行授權的動作。Controller-Manager 的 Config 有參數可以指定 `ca.crt` 跟 `ca.key` 存放的位置。
```
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```
### 7.1. Certificates API 使用
#### 7.1.1. User 建立 Key 及 CSR
```
openssl genrsa -out [jane.key ]2048
openssl req -in [jane.key] -subj "[/CN=jane]" -out [jane.csr]
```
#### 7.1.2. Admin 授權 User CSR
**Step 1: 新增 `CertificateSigningRequest` YAML**

`csr.yaml`
```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  request: [csrBase64Encode]
```
- `[csrBase64Encode]`
  
  ```
  cat jane.csr | base64 | tr -d "'n"
  ```
**Step 2: 建立 CSR Object**
```
kubectl create csr -f csr.yaml
```
```
kubectl delete csr [csrName]
```
**Step 3: 透過 Certificate API 授權或拒絕 CSR**
```
kubectl certificate approve [csrName]
```
```
kubectl certificate deny [csrName]
```
#### 7.1.3. 檢視 CSR
```
kubectl get csr
```
```
kubectl get csr jane -o yaml
```
```yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  creationTimestamp: 2019-02-13T16:36:43Z
  name: new-user
spec:
  groups:
  - system:masters
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  username: kubernetes-admin
status:
  certificate: [csrBase64Encode]
  conditions:
  - lastUpdateTime: 2019-02-13T16:37:21Z
    message: This CSR was approved by kubectl certificate approve.
    reason: KubectlApprove
    type: Approved
```
- `[csrBase64Encode]`
  
  ```
  echo [csrBase64Encode] -d
  ```

## 8. KubeConfig
為了避免每一次存取 Kubernetes 各個 Server 都要提供 CA Cert、Client Cert 以及 Client Key (如下)，可以把資訊都集中放在 KubeConfig 中。
```
kubectl get pods \
--server my-kube-playground:6443 \
--client-key admin.key \
--client-certificate admin.crt \
--certificate-authority ca.crt
```
只需要把檔案放在 `~/.kube/config/` 下就好 (不用 Create Object)。

`my-kube-playground.yaml`
```yaml
apiVersion: v1
kind: Config
current-context: my-kube-admin@my-kube-playground
clusters:
- name: my-kube-playground
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://my-kube-playground:6443
contexts:
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
    namespace:: finance
users:
- name: my-kube-admin
  user:
    client-certificate: /etc/kubernetes/pki/users/admin.crt
    client-key: /etc/kubernetes/pki/users/admin.key
```
- `certificate-authority` 也可以改用 `certificate-authority-data`，把 `crt` 以 `base64` 編碼後貼上。

```
kubectl config view (--kubeconfig=[my-kube-playground])
```
```
kubectl config current-context (--kubeconfig=[my-kube-playground])
```
```
kubectl config use-context [prod-user@production]
```

## 9. Kubernetes API Groups
### 9.1. Core
```
/api
└── /v1
    ├── namespaces
    ├── pods
    ├── rc
    ├── events
    ├── endpoints
    ├── nodes
    ├── bindings
    ├── pv
    ├── pvc
    ├── configmaps
    ├── secrets
    └── services
```

### 9.2. Named
```
/apis
├── /apps
│   └── /v1
│       ├── /deployments <-- Resources
│       │   ├── list <-- Verbs
│       │   ├── get
│       │   ├── create
│       │   ├── delete
│       │   ├── update
│       │   └── watch
│       ├── /replicasets
│       └── /statefulsets
├── /networking.k8s.io
│   └── /v1
│       └── /networkpolices
├── /certificates.k8s.io
│   └── /v1
│       └── /certificatesigningrequests
└── ...
```
> Kubernetes API Reference Doc: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/

## 10. Authorization
設定在 Kube-apiserver 的 `--authorization-mode`。
1. **Node Authorizer**
CN 名為 `system:node:[nodeName]` Group 為 `system:nodes` 都會被 Node Authorizer 授予權限。
2. **Attribute-based Access Control (ABAC)**
為每一個 User 或 User group 設立對應的權限。
3. **Role-based Access Control (RBAC)**
設定角色權限，可以綁定到 User 上。
4. **Webhook**
利用第三方軟體 (如: Open Policy Agent)。
5. **AlwaysAllow**
不確認權限，接受所有請求。
6. **AlwaysDeny**
不確認權限，拒絕所有請求。

一次設定多種 Mode: `--authorization-mode=Node, RBAC, Webhook`，會依序檢視授權，如果 `Node` 沒過，就會跳到 `RBAC`，獲得權限後，User 就得到授權。

## 11. Role-based Access Control (RBAC)
### 11.0. List API Resources
```
kubectl api-resources
```
### 11.1. Create Role
`developer-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resource: ["pods"]
  verbs: ["list", "aet", "create", "update", "delete"]
  resourceName: ["blue", "orange"]  # 只能 access 名稱為 blue 跟 orange 的 Pods
- apiGroup: [""]
  resource: ["ConfigMap"]
  verbs: ["create"]
```
- `apiGroups`: 如果是 Core Group 裡的，可以 `[""]` 表示。
```
kubectl create -f [developer-role.yaml]
```
```
kubectl get roles
```
```
kubectl describe roles [developer]
```
### 11.2. Create RoleBinding
把 Role 綁到 User 上。

`devuser-developer-binding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
subjects:
- kind: User
  name: dev-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```
- `subjects`: User 詳情 (KubeConfig 裡的 `users`)。
- `roleRef`: Role 詳情。
```
kubectl create -f [devuser-developer-binding.yaml]
```
```
kubectl get rolebindings
```
```
kubectl describe rolebinding [devuser-developer-binding]
```
### 11.3. Check Authorization
```
kubectl auth can-i [create] [deployments] (--namespace [test]) (--as [dev-user])
```
```
kubectl auth can-i [delete] [nodes] (--namespace [test]) (--as [dev-user])
```

## 12. ClusterRole and Role Binding
Namespaced vs Cluster-wide

![](https://i.imgur.com/L9haoET.png)

### 12.1. Create ClusterRole
`cluster-admin-role.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  names: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "get", "create", "delete"]
```
```
kubectl create -f [cluster-admin-role.yaml]
```

### 12.2. Create ClusterRole Binding
`cluster-admin-role-binding.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io/v1
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io/v1
```
```
kubectl create -f cluster-admin-role-binding.yaml
```

## 13. ServiceAccounts
第三方應用存取 Kubernetes API。
### 13.1. List ServiceAccount
```
kubectl get sa
```
### 13.2. Create ServiceAccount
新增 ServiceAccount。會在 `/var/rbac/` 新增 `dashboard-sa-role-binding.yaml` 及 `pod-reader-role.yaml`。
```
kubectl create sa dashboard-sa
```
為 ServiceAccount 產生 Token。
```
kubectl create token dashboard-sa
```
### 13.3. Get Service Account Details
```
kubectl describe sa dashboard-sa
```
```
Name:                dashboard-sa
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   <none>
Tokens:              <none>
Events:              <none>
```
### 13.4. Use Service Account
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-kubernetes-dashboard
spec:
  containers:
  - name: my-kubernetes-dashboard
    image: my-kubernetes-dashboard
  serviceAccountName: dashboard-sa
```
- `serviceAccountName`
- `automountServiceAccountToken: False`: Pod 在建立時不會自動 Mount `default` Service Account

若第三方應用是起在 Kubernetes 上，也可以直接把 Secret Mount 到 Pod 裡面。

## 14. Image Security
```
kubectl create secret docker-registry [secretName] \
--docker-server=[private-registry.io] \
--docker-username=[registry-user] \
--docker-password=[regitry-password] \
--docker-email=[registry-user@org.com]
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registry.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
```

## 15. Security in Docker
### 15.1. Namespace
Docker 將 Container 運行在 Host Kernel，以 Namespace 做區隔。
- 在 Container Terminal 下 `ps aux` 只會看到該 Container 執行過的 Process
- 在 Host Terminal 下 `ps aux` 可以看到在 Host 上執行的所有 Container 的 Process

Docker 預設用 root 執行所有 Process，如果要用其他身份執行，可以
1. `docker run` 建立新的 Container 時指定 `--user`
2. 建立 Docker Images 時在 Dockerfile 指定 `USER`

### 15.2. Linux Capabilities
Docker container 的 root 權限有做過調整，預設取消 `MAC_ADMIN`, `BROADCAST`, `NET_ADMIN`, `SYS_ADMIN` 等等的能力，只允許內定義的 `/usr/include/linux/capability.h` 的項目。

#### 15.2.1. 增加特定 Capabilities
```
docker run --cap-add [MAC_ADMIN] [nginx]
```
#### 15.2.2. 移除特定 Capabilities
```
docker run --cap-drop [MAC_ADMIN] [nginx]
```
#### 15.2.3. 允許所有 Capabilities
```
docker run --privileged [nginx]
```

## 16. Security Contexts
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  runAsUser: 1010
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["MAC_ADMIN"]
```
- `securityContext` 如果在 Pod 跟 Container 都有設定，Container 內部的設定會覆蓋掉 Pod 的。
- `capabilities` 只能在 Container 裡設定。

## 17. Network Policy
Kubernetes 預設 `AllAllow` 允許所有的 Pod 間都能互相溝通，如果要只允許特定的 Network Traffic (Ingress 或 Egress) 進到 Pod 裏面，就要設定 Network Policy。
> Network Solution 像是 Flannel 不支援 Network Policy。

### 17.1. Create Network Policy
`policy-definition.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
    ports:
    - protocol: TCP
      port: 3306
   egress:
   - to:
     - ipBlock:
         cidr: 192.168.5.10/32
      ports:
      - protocol: TCP
        port: 80
```
- `spec.policyTypes`: 欲限制的 Policy。如果只有定義 `Ingress`，所有的 Egress Traffic 都不會受到影響。
- `spec.ingress.from.podSelector`: 符合 `matchLabels` 的所有 Pod。
- `spec.ingress.from.namespaceSelector`: 符合 Namespace 下的所有 Pod。
- `spec.ingress.from.ipBlock`: 符合 CIDR 定義範圍的 IP。
```
kubectl create -f policy-definition.yaml
```
#### 17.1.1. Rules Setting
#### (`podSelector` AND `namespaceSelector`) OR `ipBlock`
阻擋 IP 範圍為 `192.168.5.10/32` 的 Ingress 到 `prod` Namespace 下的 `api-pod`  Pod。
```yaml
...
ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
      namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
...
```
#### `podSelector` OR `namespaceSelector` OR `ipBlock`
阻擋 IP 範圍為 `192.168.5.10/32` 的 Ingress 到 `api-pod` Pod 或是 `prod` Namespace 下的任何 Pod。
```yaml
...
ingress:
  - from:
    - podSelector:
        matchLabels:
          name: api-pod
    - namespaceSelector:
        matchLabels:
          name: prod
    - ipBlock:
        cidr: 192.168.5.10/32
...
```


---
*Written by Wan-Yu Liao; First version: December 1, 2022*

###### tags: `Kubernetes`