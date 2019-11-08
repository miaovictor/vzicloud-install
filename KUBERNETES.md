# Kubernetes 安装说明
## 安装准备
首先默认你已经安装了docker
- 关闭swap
```
swapoff -a
```
注释掉 /etc/fstab 中的所有swap空间并重启

- 安装 CURL 和 HTTPS 支持
```
sudo apt-get install apt-transport-https \
  ca-certificates \
  curl \
  software-properties-common
  ```
- 导入密钥
```
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
```
可以使用来检查是否导入成功
导入成功后，使用`sudo apt-key fingerprint 0EBFCD88`所查询到的key值应该为
```
9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88
```
- 添加软件源（阿里云）
```
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-$(lsb_release -cs) main"
```
## 安装
- 更新并安装（统一使用版本 5:19.03.4~3-0~ubuntu-bionic ）
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```
# 建立一个简单集群
## 初始化
```
sudo kubeadm init \
    --image-repository registry.aliyuncs.com/google_containers \
    --pod-network-cidr=10.244.0.0/16

# --image-repository 由于官方google_containers国内无法访问，k8s在1.13.1添加这个选项允许用户自己设置镜像获取仓库
# --pod-network-cidr 集群内部网络的网段，不应该与宿主机网络冲突。由于使用flannel作为网络插件，所以这里使用10.244.0.0/16
```
当出现以下文字，代表初始化已经完成
```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.101:6443 --token d2h9tt.l4iwvgfj85km3v7v \
    --discovery-token-ca-cert-hash sha256:c7aa1493fd8badab7b06f4765a44acc5d9afc3063b9e6b805a325ae000333ecf
```
如上文所示，如果要使用节点，必须先复制权限设置，如果重新初始化，也需要重新复制权限设置
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
请注意保存，其他节点加入集群时，直接以root权限执行这段代码即可。
```
kubeadm join 192.168.0.101:6443 --token d2h9tt.l4iwvgfj85km3v7v \
    --discovery-token-ca-cert-hash sha256:c7aa1493fd8badab7b06f4765a44acc5d9afc3063b9e6b805a325ae000333ecf
```
但是这个token只有24小时的有效期，超过时间后或者不慎忘记，可以使用 `kubeadm token create` 自己手动生成新的token

## 状态检查
我们先查看初始化好的这个管理节点状态
```
~/$ kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
vzenith-laptop   NotReady   master   15m   v1.16.2
```
可以看到，这个管理节点的状态是NotReady，现在通过详细描述去查找未准备完成的原因
```
~/$ kubectl describe nodes vzenith-laptop
...
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Fri, 08 Nov 2019 16:33:28 +0800   Fri, 08 Nov 2019 16:16:19 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 08 Nov 2019 16:33:28 +0800   Fri, 08 Nov 2019 16:16:19 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Fri, 08 Nov 2019 16:33:28 +0800   Fri, 08 Nov 2019 16:16:19 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Fri, 08 Nov 2019 16:33:28 +0800   Fri, 08 Nov 2019 16:16:19 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
...
```
正常情况下MemoryPressure、DiskPressure和PIDPressure都应该为False，表示内存、磁盘和PID充足；Ready为True，表示准备就绪

这里Ready为False的原因是NetworkPluginNotReady，网络插件未准备就绪。

### 删除污点
由于kubernetes中的管理节点默认是不能运行Pod的，因为默认管理节点默认会有一个污点（Taints）。所以安装网络插件之前，我们需要先删除节点上的污点

回到刚刚的详细描述中，有这样一条
```
Taints:             node-role.kubernetes.io/master:NoSchedule
```
通过 `kubectl taint nodes --all node-role.kubernetes.io/master-` 来删除这个污点

### 安装网络插件
我们选择的网络插件为flannel，他官方的配置文件中的镜像在国内无法下载到。有两个解决办法：
- 从加速器处下载镜像，并修改tag
```
docker pull quay.azk8s.cn/coreos/flannel
docker tag <刚刚下载的镜像ID> -t quay.io/coreos/etcd
```
然后应用配置
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
- 修改配置文件中的镜像

下载 [kube-flannel.yml](https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml)

将文件中
quay.io/coreos/flannel:版本
改为
quay.azk8s.cn/coreos/flannel:版本
然后应用配置
```
kubectl apply -f <kube-flannel.yml路径>
```

### 结束
查看节点状态
```
~/$ kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
vzenith-laptop   Ready   master   110m   v1.16.2
```