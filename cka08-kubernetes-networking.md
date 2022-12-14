# CKA08- Kubernetes Networking
## 1. Networking Prerequisites
- **Switch**: 連結兩個系統的網路介面 (Network Interface, 如 eth0)。
- **Router**: 連結兩個不同的網路，使其能夠溝通。
- **Gateway**: 通往外部 Internet 的門。
### 1.1. Commands
#### 1.1.1. Interfaces, IP Address and MAC Address
```
# list interfaces
ip link

# list ip address and mac address on the interfaces
ip addr
if config

# set ip address to the interfaces
ip addr add [192.168.1.11/24] dev [eth0]

# set ip address to the interfaces (permanently)
## setup in `/etc/network/interfaces`
```
#### 1.1.2. Routing
![](https://i.imgur.com/hfBkoKm.png)
```
# list routing config
ip route
route

# add gateway
ip route add [192.168.1.0/24] via [192.168.2.1]
ip route add [default] via [192.168.2.1]
ping 192.168.1.10  # ping `102.168.1.10` from `192.168.2.10`
```
#### 1.1.3. Linux Forward Packets
上面 `ping 192.168.1.10` 還不會有 Return。因為以 Linux 作為 Router 時，預設無法從一網路介面 Forward Packets 到另一個網路介面。
```
# allow forwarding packets
cat /proc/sys/net/ipv4/ip_forward  # 0: Forward packets not allowed; 1: Forward packets allowed
echo 1 > /proc/sys/net/ipv4/ip_forward

# allow forwarding packets (permanently)
## setup in `/etc/sysctl.conf`
```
`/etc/sysctl.conf`
```
...
net.ipv4.ip_forward = 1
...
```
### 1.2. Network Namespaces
#### 1.2.1. Commands
```
# create network namespace
ip netns add [red]
ip netns add [blue]

# list network namespace
ip netns

# list interfaces
ip netns exec [red] ip link
ip -n [red] link

# list routing table
ip netns exec [red] route

# list arp table
ip netns exec [red] arp
```
- Address Resolution Protocol (ARP) Table: 記錄 IP 對應的 MAC Address 的清單。
#### 1.2.2. Communicate Two Network Namespace
![](https://i.imgur.com/31aruQ2.png)
```
# 1. create veth
ip link add [veth-red] type veth peer name [veth-blue]

# 2. attach veth to network namespaces
ip link set [veth-red] netns [red]
ip link set [veth-blue] netns [blue]

# 3. assign ip to network namespaces
ip -n [red] add [192.168.15.1] dev [veth-red]
ip -n [blue] add [192.168.15.2] dev [veth-blue]

# 4. bring up interface
ip -n [red] link set [veth-red] up
ip -n [blue] link set [veth-blue] up

# 5. test
ip -n [red] ping [192.168.15.2]  # ping `blue` from `red`
ip -n [red] arp  # red arp table 有 blue 的資訊

# 6. delete veth
ip -n [red] link del [veth-red]
```
#### 1.2.3. Communicate Multiple Network Namespace with Bridge
虛擬 Switch (Bridge) 如 Linux Bridge, Open vSwitch。

![](https://i.imgur.com/9lQrdP4.jpg)
```
# 1. create veth and bridge
ip link add [v-net-0] type bridge
ip link set dev [v-net-0] up
ip link add [veth-red] type veth peer name [veth-red-br]
ip link add [veth-blue] type veth peer name [veth-blue-br]

# 2. attach veth and bridge to network namespaces
ip link set [veth-red] netns [red]
ip link set [veth-red-br] master [v-net-0]
ip link set [veth-blue] netns [blue]
ip link set [veth-blue-br] master [v-net-0]

# 3. assign ip to network namespaces
ip -n [red] add [192.168.15.1] dev [veth-red]
ip -n [blue] add [192.168.15.2] dev [veth-blue]

# 4. bring up interface
ip -n [red] link set [veth-red] up
ip -n [blue] link set [veth-blue] up

# 5. connect host and bridge
ip addr add [192.168.15.5/24] dev [v-net-0]

# 6. set host as the gateway
ip -n [blue] ip route add [192.168.1.0/24] via [192.168.15.5]

# 7. add rule to nat ip table
iptables -t nat -A POSTROUTING -s [192.168.15.0/24] -j MASQUERADE
## 把從 `192.168.15.0/24` 出去的 Network Traffic 改寫為從 `192.168.1.2` 出去的。

# 8. add `default` to routing table
ip -n [blue] ip route add default via [192.168.15.5]

# 9. port forwarding
iptables -t nat -A PREROUTING --dport [80] --to-destination [192.168.15.2:80] -j DNAT
```

### 1.3. Docker Networking
```
docker run --network [none|host|bridge] [nginx]
```
- `--network bridge`: Default，Docker 會在 Host 上建立名為 `docker0` 的 Bridge。
#### How it works?
![](https://i.imgur.com/4icZOqt.jpg)

Container 建立後，會看到一個新的 Network Namespace 被建立。
```
ip netns
ip link
ip -n [b3165c10a92b] link
iptables -nvL -t nat  # port forwarding
```

### 1.4. Container Networking Interface (CNI)
Docker Bridge 處理 Network 的步驟:
```
1. Create Network Namespace
2. Create Bridge Network/Interface
3. Create veth Pairs (Pipe, Virtual Cable)
4. Attach veth to Namespace
5. Attach Other veth to Bridge
6. Assign IP Addresses
7. Bring the interfaces up
8. Enable NAT – IP Masquerade
```
- Docker 之外的其他 Container Solution (如 rkt, MESOS) 也用類似當方式處理 Container Network
- 因此訂定 CNI 就可以透過一套統一的 Container Runtime 跟 Plugins 標準使用類似下面的一行指令完成 Container Network Configuration。
  ```
  bridge add 2e34dcf34 /var/run/netns/2e34dcf34
  ```
- Docker 沒有用 CNI，Docker 使用類似於 CNI 的 Container Network Model (CNM)。
### 1.5. DNS
#### 1.5.1. Domain Names
```
www.kubernetes.io
|   |          |
Subdomain      |
    |          |
    Top-level Domain (TLD)
               |
               Root
```
1. `Root`: Root Server 解析 `.com`, `.edu`, `.org`, `.io` 等等。
2. `Top-level Domain (TLD)`: TLD Server 解析 `kubernetes`。
3. `Subdomain`: Authoritative Server 解析 `www`。
#### 1.5.2. Record Types
- **A Record**: IPv4 Address，將 Domain Name 對應到 IPv4 Address。
- **AAAA Record**: IPv6 Address，將 Domain Name 對應到 IPv6 Address。
- **CNAME Record**: Canonical Name，將不同別名對應到同一部主機。
#### 1.5.3. DNS Setting
**`/etc/hosts`**: 設定 Name Resolution
```
127.0.0.1       localhost
10.32.1.2       app
```
**`/etc/resolv.conf`**: 設定 DNS Server IP、Search Domain
```
nameserver 10.96.0.10  # dns server ip address
search default.svc.cluster.local svc.cluster.local cluster.local  # search domain
options ndots:5
```
**`/etc/nsswitch.conf`**: 設定 Resolve 優先順序
```
...
hosts:     files dns  # 先查 `/etc/hosts`，再查 dns host
...
```
#### 1.5.4. DNS Command
**`nslookup`**
```
nslookup [www.google.com]
```
**`dig`**
```
dig [www.google.com]
```
:::info
`nslookup` 跟 `dig` 都只查看 DNS Server，沒有納入 `/etc/hosts` 裡的 Domain Name。
:::
### 1.6. CoreDNS Setup
以 CoreDNS 將 Host 設為 DNS Server。
#### 1.6.1. Install CoreDNS
```
wget https://github.com/coredns/coredns/releases/download/v1.10.0/coredns_1.10.0_linux_amd64.tgz
tar -xzvf coredns_1.10.0_linux_amd64.tgz
./coredns
```
#### 1.6.2. Specify IP/Hostname Mapping
`Corefile`
```
. {
  hosts /etc/hosts
}
```

## 2. Cluster Networking
#### Control Plane Required Ports
![](https://i.imgur.com/E7veeXx.png)

#### Worker Nodes Required Ports
![](https://i.imgur.com/uNQZZzV.png)

#### Get Ports
```
netstat -tulp
```

## 3. Pod Networking
![](https://i.imgur.com/Iev86Um.jpg)

![](https://i.imgur.com/uSAz9tk.jpg)

### 3.1. CNI in Kubernetes
#### 3.1.1. CNI Configuration
由於 Pod 是由 Kubelet 負責創建，因此 CNI 的設定也是在 Kubelet。
```
ps aux | grep --color kubelet
```
```!
root        2330  0.0  0.0 4226660 98040 ?       Ssl  04:20   0:24 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runtime=remote --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock --pod-infra-container-image=k8s.gcr.io/pause:3.7
```
- `--network-plugin=cni`: Network Plugin 種類。
- `--cni-conf-dir=/etc/cni/net.d`: CNI Configuration File 位置。
- `--cni-bin-dir=/opt/cni/bin`: 預設 CNI 執行檔位置。`ls /opt/cni/bin` 可以看 Host 上可用的 CNI Plugin。
### 3.2. CNI Weave
在 Packet 外再多包裹一層，運送到目標 Node 後再拆開。

![](https://i.imgur.com/Rlsa250.jpg)
#### Deploy Weave
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Deployed as DaemonSet
> https://v1-22.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node
### 3.2. IP Address Management (IPAM)
Weave 會將預設 Pod IP 範圍為 `10.32.0.0/12` (`10.32.0.1`~`10.32.255.254`) 平均分配到各個 Node。
#### 查看 POD 可用 IP 範圍
```
kubectl logs -n kube-system [weave-net-7dml7]
```
#### 設定 POD 可用 IP 範圍
修改 `/etc/cni/net.d/net-script.conf` 裡的 `ipam`。
```yaml
{
  "cniVersion": "0.2.0",
  "name": "mynet",
  "type": “net-script",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.0.0/16",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

## 4. Service Networking
Service 建立時會被 Assign IP，**Kube-proxy 得知新建立的 Service IP Address 後會在 Cluster 的每一個 Node 上新增 Forwarding Rules**，把送到 Service 的 Traffic 全部 Forward 到對應的 Pod。
### 4.1. 設定 Forwarding Rules
設定 Kube-proxy 時指定`--proxy-mode [Mode]`。
- `iptables`: Default
- `userspace`
- `ipvs`
#### 查看 Forwarding Rules
```
kubectl logs -n kube-system [kube-proxy-m9xpb] | grep --color proxyMode
```
```
cat /var/log/kube-proxy.log
```
### 4.2. 設定 ClusterIP 範圍
設定 Kube-apiserver 時指定 `--service-cluster-ip-range [ipRange]`。
- `10.0.0.0/24`: Default
### 4.3. 查看由 Kube-proxy 建立的 IP Table Rules
```
iptables -L -t nat | grep nginx
```
```
KUBE-MARK-MASQ             all  --  10.50.192.1          anywhere             /* default/nginx:8080-8080 */
DNAT                       tcp  --  anywhere             anywhere             /* default/nginx:8080-8080 */ tcp DNAT [unsupported revision]
KUBE-SVC-3ZVO6MMCRQUXYOC5  tcp  --  anywhere             10.105.131.148       /* default/nginx:8080-8080 cluster IP */ tcp dpt:http-alt
KUBE-MARK-MASQ             tcp  -- !10.244.0.0/16        10.105.131.148       /* default/nginx:8080-8080 cluster IP */ tcp dpt:http-alt
KUBE-SEP-PINH4Z4JZXK5ZU4X  all  --  anywhere             anywhere             /* default/nginx:8080-8080 -> 10.50.192.1:8080 */
```
代表任何到 `nginx` Service (`10.105.131.148:8080`) 的 Traffic 都會被 Forward 到 `nginx` Pod (`10.50.192.1:8080`)。
```
kubectl get pod -o wide
```
```
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE     NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          15m   10.50.192.1   node01   <none>           <none>
```
```
kubectl get svc
```
```
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    45m
nginx        ClusterIP   10.105.131.148   <none>        8080/TCP   8m37s
```

## 5. DNS in Kubernetes
### 5.1. Domain Name of Pod and Service
![](https://i.imgur.com/pyC7jmd.jpg)

### 5.2. CoreDNS in Kubernetes
- 通常以 `Deployment` 部署「`coreDNS` `Pod`」，並建立「`kube-dns` `Service`」。
- 每新增一個 Pod，Kubelet 就會自動將 Kubelet Config (`/var/lib/kubelet/config.yaml`) 裡設定的 `clusterDNS` IP 設為 `/etc/resolv.conf` 的 `nameserver`。
#### 5.2.1. Corefile
`/etc/coredns/Corefile` 會以 `Configmap` 形式 Mount 到 `CoreDNS` `Pod`。
```
kubectl get cm [coredns] -n kube-system
```
```
.:53 {
  errors
  health
  kubernetes cluster.local in-addr.arpa ip6.arpa {
    pods insecure
    upstream
    fallthrough in-addr.arpa ip6.arpa
  }
  prometheus :9153
  proxy . /etc/resolv.conf
  cache 30
  reload
}
```
- `Kubernetes` Plugin 使 CoreDNS 能夠運用在 Kubernetes。
    - `cluster.local` 為 Root Domain Name。
- `proxy`: 無法被 DNS Server 解析的 Record (如 `www.google.com`) 會被 Forward 到 `/etc/resolv.conf` 的 Name Server。
#### 5.2.2. 查詢 Service 的 Full Domain Name
```
host [serviceName]
nslookup [serviceName]
```
- `/etc/resolv.conf` 有 `search` Entry，所以只要輸入 Service Name，就會自動補上 `search` Entry 內的其他 Subdomain Name, Root Domain Name。

## 6. Ingress
### 6.1. Service LoadBalancer vs Ingress
#### 6.1.1. Service LoadBalancer
![](https://i.imgur.com/f9jRWnC.png)
#### 6.1.2. Ingress
Ingress 依據 URL Path 將 Traffic 導到不同的 Service 並使用 SSL。
![](https://i.imgur.com/NkjS6U6.png)
- Deploy Nginx -> Deploy Ingress Controller
- Config Path -> Create Ingress Resources
### 6.2. Deploy Ingress Controller
Kubernetes Project 維護的 Ingress Controller:
- GCP HTTP(S) Load Balancer (GCE)
- Nginx
#### 6.2.1. Create Namespace for Ingress
```
kubectl create ns ingress-space
```
#### 6.2.2. Create a ConfigMap
```
kubectl create cm nginx-configuration -n ingress-space
```
#### 6.2.3. Create a ServiceAccount
```
kubectl create sa ingress-serviceaccount -n ingress-space
```
#### 6.2.4. Create a Role and a RoleBinding for ServiceAccount

#### 6.2.5. Create a Deployment of Ingress Controller
`ingress-controller.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-controller
  namespace: ingress-space
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      serviceAccountName: ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --default-backend-service=app-space/default-http-backend
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
```
```
kubectl create -f ingress-controller.yaml
```
#### 6.2.6. Create a Service to make Ingress available to external users
```
kubectl expose deployment -n ingress-space ingress-controller --type=NodePort --port=80 --name=ingress --dry-run=client -o yaml > ingress-svc.yaml
```
`ingress-svc.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress
  namespace: ingress-space
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 30080
  selector:
    name: nginx-ingress
  type: NodePort
```

### 6.3. Create an Ingress
#### 6.3.1. `www.my-online-store.com`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wear
spec:
  backend:
    service:
      name: wear-service
      port:
        number: 8282
```
#### 6.3.2. `www.my-online-store.com` + `/wear`, `/watch`
```
kubectl create ing [ingress-wear-watch] --rule=[/wear=wear-service:8080] --rule=[/watch=video-service:8080] -n [app-space] > ingress-wear-watch.yaml
```
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
  name: ingress-wear-watch
  namespace: app-space
spec:
  rules:
  - http:
      paths:
      - path: /wear
        pathType: Prefix
        backend:
          service:
            name: wear-service
            port:
              number: 8282
      - path: /watch
        pathType: Prefix
        backend:
          service:
            name: watch-service
            port:
              number: 8080
```
- `rewrite-target: /`: Rewirte URLs by replacing whatever is under `spec.rules.http.paths.path`
    - With `rewrite-target`:
        - `http://<ingress-service>:<ingress-port>/watch` 
        - `http://<watch-service>:<port>/`
    - Without `rewrite-target`:
        - `http://<ingress-service>:<ingress-port>/watch`
        - `http://<watch-service>:<port>/watch` <-- 404
- `ssl-redirect: "false"`: By default the controller redirects (308) to HTTPS if TLS is enabled for that ingress
#### 6.3.3. `wear.my-online-store.com`, `watch.my-online-store.com`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
  name: ingress-wear-watch
  namespace: app-space
spec:
  rules:
  - host: wear.my-online-store.com
    http:
      paths:
      - backend:
          service:
            name: wear-service
            port:
              number: 8282
   - host: wear.my-online-store.com
     http:
       paths:
       - backend:
           service:
            name: watch-service
            port:
              number: 8080
```
```
kubectl create -f [ingress.yaml]
```
> `nginx.ingress.kubernetes.io`: https://kubernetes.github.io/ingress-nginx/examples/

---
*Written by Wan-Yu Liao; First version: December 11, 2022*

###### tags: `Kubernetes`