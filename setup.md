# 1.环境

| 主机名称   | IP地址         | 说明       | 软件                                                         |
| ---------- | -------------- | ---------- | ------------------------------------------------------------ |
| k8s-master | 192.168.43.230 | master节点 | kube-apiserver、kube-controller-manager、<br />kube-scheduler、etcd、kubelet、kube-proxy |
| k8s-node1  | 192.168.43.232 | node节点   | kubelet、kube-proxy                                          |

| 软件   | 版本  |
| ------ | ----- |
| Ubuntu | 20.04 |
|kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy|v1.22.2|
|etcd|kubeadm默认|
|docker-ce|v20.10.7|


网段

物理主机：192.168.43.0/24

service：10.96.0.0/12             kubeadm默认

pod：172.56.0.0/16                手工指定



## 1.1初始化系统操作
### 1.2.安装Vmware Tools
状态栏->虚拟机->安装Vmware Tools
```shell
cp /media/$(whoami)/'VMware Tools'/Vmware*.tar.gz ~
cd ~
tar -xf Vmware*.tar.gz
# 第一个选项是yes，其他一路回车
sudo ./vmware-tools-distrib/vmware-install.pl 
```
### 1.3.以root用户登录
```shell
sudo passwd root
sudo su -               # 终端登录root用户
```

### 1.4.配置vim
```shell
sed -e 's|set compatible|set nocompatible|g' \
         -e '/set nocompatible/aset backspace=2' \
         -i.bak \
         /etc/vim/vimrc.tiny
```

### 1.5.配置IP
虚拟机的网络模式需要是NAT
```shell
cat > /etc/netplan/01-network-manager-all.yaml <<EOF
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: no
      dhcp6: no
      addresses: [192.168.43.230/24]
      gateway4: 192.168.43.2
      nameservers:
        addresses: [114.114.114.114,8.8.8.8]
EOF

netplan apply
```
这里可以尝试重启一下主机了

### 1.6.修改仓库源，安装必要的软件
##### 以root用户登录
```shell
# 对于 Ubuntu
sed -i 's/cn.archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
apt-get update
apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

### 1.7.关闭防火墙
```shell
ufw disable
systemctl disable --now ufw
ufw status
```

### 1.8.关闭交换分区

```shell
sed -ri 's/.*swap.*/#&/' /etc/fstab
swapoff -a && sysctl -w vm.swappiness=0

cat /etc/fstab
# /swapfile                                 none            swap    sw              0       0
```

### 1.9.安装ssh server
```shell
apt install openssh-server -y
sed -ri 's/.*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
systemctl restart ssh
```

### 1.10修改内核参数
```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

### 1.11.配置hosts本地解析
```shell
cat >> /etc/hosts <<EOF
192.168.43.230 k8s-master
192.168.43.232 k8s-node1
EOF
```


# 2.k8s基本组件安装

## 2.1.安装docker
```shell
# 检查是否存在docker
dpkg -l | grep docker
# 按照阿里云文档设置docker安装源
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"

apt-get install docker-ce=5:20.10.7~3-0~ubuntu-focal -y
# 验证
docker info


tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://o752yjuw.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl daemon-reload
systemctl restart docker

```

## 2.2.安装Kubernetes
```shell
# 密钥
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
# 存储库
tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
EOF
# 安装
apt-get update
apt-get install -y kubelet=1.22.2-00 kubeadm=1.22.2-00 kubectl=1.22.2-00 
apt-mark hold kubelet kubeadm kubectl

```

## 2.3.克隆节点
到这里master和node节点该安装的东西已经到位，现在需要克隆节点来分别操作了。
可以给镜像拍一个快照，然后克隆节点出来，192.168.43.230作为master节点，那么就克隆出来的192.168.43.232作为node节点
注意是关机后拍快照，克隆是使用完整克隆。



## 还是进入刚才的master节点
## 2.4.初始化Kubernetes集群
```shell
# 设置Master主机名
hostnamectl --static set-hostname k8s-master
# 集群初始化，apiserver填自己的master节点ip，pod是自定义且不与apiserver冲突的就行
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.22.2 \
 --pod-network-cidr=172.56.0.0/16 \
 --apiserver-advertise-address=192.168.43.230


# 记录join的token，无须执行，是node节点用来执行，可以用记事本把命令记录下来。
#kubeadm join 192.168.43.230:6443 --token xrw9e1.wqv5dhharstyfaq5 \
#        --discovery-token-ca-cert-hash #sha256:6ad1563559ef87862325c7e5704dc5ca26e0e1e1286738c34b9d1056d7f6678d 
        
        
# 我们使用root用户，复制 kubeconfig配置文件
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf


# 让master也参与pod调度
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 2.5.controller-manager和scheduler启用非安全端口
```shell
sed -ri 's/^.*port=0.*/#&/' /etc/kubernetes/manifests/kube-controller-manager.yaml
sed -ri 's/^.*port=0.*/#&/' /etc/kubernetes/manifests/kube-scheduler.yaml
# 重启一下组件服务
systemctl restart kubelet.service
# 查看
kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
etcd-0               Healthy   {"health":"true","reason":""}   
controller-manager   Healthy   ok                              
scheduler            Healthy   ok     
```



## 建议在安装网络插件之前对master主机拍一次快照，以便回滚

## 2.6.安装calico网络插件
```shell
# 先把插件拉下来
docker pull quay.io/calico/cni:v3.25.0
docker pull quay.io/calico/kube-controllers:v3.25.0
docker pull quay.io/calico/node:v3.25.0
docker pull quay.io/calico/pod2daemon-flexvol:v3.25.0
docker pull quay.io/calico/csi:v3.25.0
docker pull quay.io/calico/node-driver-registrar:v3.25.0
# 创建资源对象
kubectl create -f https://projectcalico.docs.tigera.io/archive/v3.25/manifests/tigera-operator.yaml
curl -L https://projectcalico.docs.tigera.io/archive/v3.25/manifests/custom-resources.yaml -o /root/custom-resources.yaml
# 修改cidr与kubeadm init的pod-network-cidr保持一致
sed -ri 's/cidr:.*/cidr: 172.56.0.0\/16/' /root/custom-resources.yaml
kubectl create -f /root/custom-resources.yaml
```
查看网络插件资源对象的状态
```shell
kubectl get pods --all-namespaces 
NAMESPACE          NAME                                     READY   STATUS    RESTARTS      AGE
calico-apiserver   calico-apiserver-6bcccb8d9d-7m968        1/1     Pending   1 (21m ago)   27m
calico-system      calico-kube-controllers-8fdfc695-w977t   1/1     Pending   0             35m
calico-system      calico-node-lscrf                        1/1     Init:0/2   1 (21m ago)   35m
calico-system      calico-typha-677dcbbd4b-47svh            1/1     Pending   1 (21m ago)   35m
calico-system      csi-node-driver-q57jl                    2/2     Pending   1 (22m ago)   29m
···
```
如果显示``Pending``的话，让节点尝试工作

```shell
kubectl taint node k8s-master node.kubernetes.io/not-ready-
```
```shell
# 再次查看
kubectl get pods --all-namespaces
NAMESPACE         NAME                                     READY   STATUS              RESTARTS   AGE
calico-system     calico-kube-controllers-8fdfc695-w977t   0/1     ContainerCreating   0          6m
calico-system     calico-node-lscrf                        0/1     Init:0/2            0          6m
calico-system     calico-typha-677dcbbd4b-47svh            1/1     Running             0          6m1s
calico-system     csi-node-driver-q57jl                    0/2     Pending             0          2s
kube-system       coredns-7f6cbbb7b8-92k8b                 0/1     ContainerCreating   0          71m
kube-system       coredns-7f6cbbb7b8-qzbqb                 0/1     ContainerCreating   0          71m
```
如果有Pending和ContainerCreating，calico 初始化会很慢 需要耐心等待一下，大约5分钟左右，容器镜像就会部署到pod中，不知道这么说对不对。最后所有的pod都会处于``Running``
```shell
# 网络插件成功部署
kubectl get node
NAME         STATUS   ROLES                  AGE    VERSION
k8s-master   Ready    control-plane,master   111m   v1.22.2
```

## 开始进入克隆出来的node节点

注意：
* 进入node节点前master主机最好不要在线，因为我们的网卡地址还没有修改，IP地址可能会有冲突的时候，所以可以先挂起master主机。
* **以下操作没有指定那么都是在node节点来操作**

## 2.7.node节点初始化
##### 以root用户登录
```shell
# 修改IP
sed -i '0,/\(addresses: \).*/s//\1[192.168.43.232\/24]/' /etc/netplan/01-network-manager-all.yaml
# 设置node1主机名
hostnamectl --static set-hostname k8s-node1

netplan apply
```

## 2.8.node节点加入集群

```shell
# 执行2.4记录下来的join命令
kubeadm join 192.168.43.230:6443 --token xrw9e1.wqv5dhharstyfaq5 \
        --discovery-token-ca-cert-hash sha256:6ad1563559ef87862325c7e5704dc5ca26e0e1e1286738c34b9d1056d7f6678d 
        
# 拷贝配置，在master节点上，操作
ssh  192.168.43.232 mkdir -p /root/.kube/
scp /etc/kubernetes/admin.conf root@192.168.43.232:/root/.kube/config

# 查看
kubectl get nodes
NAME         STATUS     ROLES                  AGE   VERSION
k8s-master   Ready      control-plane,master   12h   v1.22.2
k8s-node1    NotReady   <none>                 74s   v1.22.2
```
可以看到node节点的网络插件还没有安装上，再等待5分钟左右node节点会自动初始化加载master组件，就会加入到集群里面的了。

```shell
kubectl get nodes
NAME         STATUS   ROLES                  AGE     VERSION
k8s-master   Ready    control-plane,master   12h     v1.22.2
k8s-node1    Ready    <none>                 8m33s   v1.22.2
```

如果还是显示``NotReady``的话，再在node节点执行``2.6``。



---

参考：[Kubernets的二进制安装](https://github.com/cby-chen/Kubernetes/blob/main/doc/v1.27.1-CentOS-binary-install-IPv6-IPv4-Three-Masters-Two-Slaves-Offline.md)