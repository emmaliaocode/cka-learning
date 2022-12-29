# CKA09- Design and Install a Kubernetes Cluster
## 1. Choosing Kubernetes Infrastructure
- Virtualization Software: Hyper-V, VMware Workstation, VirtualBox
### 1.1. Start Kubernetes on Local Machine
#### 1.1.1. For learning purpose
| Solution | Requires VMs to be ready | # of Nodes in Cluster   |
| -------- | ------------------------ | ----------------------- |
| Minikube | X                        | Single Node             |
| Kubeadm  | V                        | Single Node, Multi Node |
> Deploy Kubernetes from scratch: 
> - https://www.youtube.com/watch?v=uUupRagM7m0&list=PL2We04F3Y_41jYdadX55fdJplDvgNGENo
> - https://github.com/mmumshad/kubernetes-the-hard-way
#### 1.1.2. For productions purpose
| Type              | Provision, configure, maintain VMs by | Solutions                                                                                                                        |
| ----------------- | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Hosted            | Providers                             | Google Container Engine (GKE), OpenShift Online, Azure Kubernetes Service, Amazon Elastic Container Service for Kubernetes (EKS) |
| Turnkey Solutions | Yourself                              | Kubernetes on AWS using `kops`, OpenShift, Cloud Foundry Container Runtime, VMware Cloud PKS, Vagrant                            |
### 1.2. Configure High Availability (HA)
#### 1.2.1. Multiple Master Node
| Components         | Mode                         | How                                                                                                                                                        |
| ------------------ | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Kube-apiserver     | Active-Active                | Load-balancer (Nginx) configed in the front of Kube-apiserver                                                                                              |
| Controller-Manager | Active-Standby               | Leader-Election Process: `--leader-elect true`, `--leader-elect-lease-duration 15s`, `--leader-elect-renew-deadline 15s`, `--leader-elect-retry-period 2s` |
| Scheduler          | Active-Standby               | Leader-Election Process: `--leader-elect true`, `--leader-elect-lease-duration 15s`, `--leader-elect-renew-deadline 15s`, `--leader-elect-retry-period 2s` |
| etcd               | Stacked or External Topology | `--etcd-servers=https://[ip]:2379,https://[ip]:2379`                                                                                                       |
### 1.3. etcd in HA
- Distributed System
- Consistent: etcd 確保一筆資料在架構下的所有 Instance 上是一致的
- 於 `etcd.service` 設定 `--initial-cluster peer-1=https://[PEER1IP:2380,peer-2=https://[PEER2IP:2380`
#### 1.3.1. Write
限制只有一個 Instance 可寫入資料。以 Leader-Election 決定 Leader。Leader Node 接收到 Write 內容時寫入，並在其他 Follower Node 製作副本。
#### 1.3.2. RAFT Leader-Election
每個 Node 產生 Random Timer，最快結束 Random Timer 倒數的 Node 發送 Leader Request 給剩下的 Node，剩下的 Node 回覆後完成 Leader-Election。
#### 1.3.3. Quorum
`N/2+1` (Quorum) 個 Node 都寫入，才算資料寫入完成。
| # of etcd Node (`N`) | Quorum | Fault Tolerance |
| -------------------- | ------ | --------------- |
| 1                    | 1      | 0               |
| 2                    | 2      | 0               |
| 3 (Recomanded)       | 2      | 1               |
| 4                    | 3      | 1               |
| 5 (Recomanded)       | 3      | 2               |
| 6                    | 4      | 2               |
| 7 (Recomanded)       | 4      | 3               |
- 奇數的 `N` 於 Network Partitiion (`N` 對切之後 Quorum 都有符合) 情境下的容錯率較高。

## 2. Deployment with Kubeadm
> [Doc: Bootstrapping clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
### 2.1. Provision the VMs
```
git clone https://github.com/emmaliaocode/vagrant-vmware-macos-arm.git
cd vagrant-vmware-macos-arm
vagrant status
vagrant up
vagrant ssh [hostname]
```
### 2.2. Host setup
```
sudo -i
```
```
swapoff -a
vim /etc/hosts
```
### 2.3. Install container runtime
#### 2.3.1. Forwarding IPv4 and letting iptables see bridged traffic
[Doc](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
```
lsmod | grep br_netfilter
```
```
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
#### 2.3.2. Install Docker Engine
[Doc](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
```
sed -i 's/disabled_plugins = \[\"cri\"\]/\#disabled_plugins \= \[\"cri\"\]/g'  /etc/containerd/config.toml
```
```
systemctl restart containerd
systemctl status containerd
```
### 2.4. Installing kubeadm, kubelet and kubectl
[Doc](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
```
# set container runtime endpoint for `crictl`
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
```
~~kubelet -v=5 \
--container-runtime=remote \
--container-runtime-endpoint=/var/run/containerd/containerd.sock \
--resolv-conf=/run/systemd/resolve/resolv.conf~~
```

### 2.5. Configuring a cgroup driver
Configuring cgroup driver is not applicable when using Docker as kubeadm will automatically detect the group driver for the kubelet.
### 2.6. Initiate controlplane
```
kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.6.138

# kubeadm reset
```
```
...
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.6.136:6443 --token am4pwi.oc5w6wneglszn7ob \
	--discovery-token-ca-cert-hash sha256:00450443a39276a758fa72e20f72619b7dcc84aff241f624e59f091fdb527290
```
```
sudo -i
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc
source ~/.bashrc
```
```
kubectl get nodes
```
```
NAME         STATUS   ROLES           AGE   VERSION
kubemaster   Ready    control-plane   17m   v1.26.0
```
```
kubectl get pod -A
```
```
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-787d4945fb-b8xs7             0/1     Pending   0          94s
kube-system   coredns-787d4945fb-lj7lm             0/1     Pending   0          94s
kube-system   etcd-kubemaster                      1/1     Running   0          109s
kube-system   kube-apiserver-kubemaster            1/1     Running   0          110s
kube-system   kube-controller-manager-kubemaster   1/1     Running   0          109s
kube-system   kube-proxy-k76mp                     1/1     Running   0          94s
kube-system   kube-scheduler-kubemaster            1/1     Running   0          109s
```
### 2.7. Installing a Pod network add-on
List the avaliable network plugins on host.
```
ls /opt/cni/bin
```
https://kubernetes.io/docs/concepts/cluster-administration/addons/
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml
```
### 2.8. Possiable Issue
#### 2.8.1. Flannel: Add `/run/flannel/subnet.env` to each node
```
vim /run/flannel/subnet.env
```
```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
#### 2.8.2. Flannel: Remove `resources` setting of flannel daemonset
```
kubectl patch daemonset -n kube-flannel kube-flannel-ds --type=json -p='[{"op": "remove", "path": "/spec/template/spec/containers/0/resources"}]'
```
#### 2.8.3. Flannel: "failed to set bridge addr: "cni0" already has an IP address different from 10.244.1.1/24"
[Link](https://blog.csdn.net/Wuli_SmBug/article/details/104712653)

### 2.9. Join worker nodes
```
sudo su
kubeadm join 192.168.6.136:6443 \
--token am4pwi.oc5w6wneglszn7ob \
--discovery-token-ca-cert-hash sha256:00450443a39276a758fa72e20f72619b7dcc84aff241f624e59f091fdb527290
```
```
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
### 2.10. Regenerate apiserver certs (if reboot)
Did not specify static IP, regenerate certs for apiserver
```
rm /etc/kubernetes/pki/apiserver.*
kubeadm init phase certs apiserver --apiserver-advertise-address 192.168.6.140
```
### 2.11. Reference
- [arm64部署k8s](http://liupeng0518.github.io/2019/03/20/k8s/deploy/arm64%E9%83%A8%E7%BD%B2k8s/)

## 3. End-to-end Tests on a Kubernetes Cluster
> [Youtube: Install Kubernetes from Scratch [19] - End to End Tests](
https://www.youtube.com/watch?v=-ovJrIIED88&list=PL2We04F3Y_41jYdadX55fdJplDvgNGENo&index=19)


---
*Written by Wan-Yu Liao; First version: December 14, 2022*

###### tags: `Kubernetes`