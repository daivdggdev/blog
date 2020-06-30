---
title: k8s部署篇
date: 2020-06-30 10:53:55
tags: k8s
---

## Centos

参考这边文章即可，从部署到验证

[使用kubernetes 官网工具kubeadm部署kubernetes(使用阿里云镜像)](https://www.cnblogs.com/tylerzhou/p/10971336.html)


## Raspberry Pi 4 cluster

### 引言

这里主要参考了下面的教程：

* https://gist.github.com/aaronkjones/d996f1a441bc80875fd4929866ca65ad
* https://github.com/alexellis/k8s-on-raspbian/blob/master/GUIDE.md

### 系统准备

这里的准备系统是要准备树莓派操作系统，一般来讲运行kubernetes这么大的系统，最好的系统选择方式为轻量级的操作系统，这里推荐操作系统：

* Raspbian Stretch Lite - https://www.raspberrypi.org/downloads/raspberry-pi-os/

> 官方系统的最大好处是软件支持的源比较丰富，不要尝试诸如`Centos`等在树莓派上偏冷门的系统，因为国内源很少

### 树莓派系统安装

在Macos上的安装比较简单，下载完镜像后用 `balenaEtcher` 烧录到SD卡中，再根据教程[无屏幕和键盘配置树莓派WiFi和SSH](https://shumeipai.nxez.com/2017/09/13/raspberry-pi-network-configuration-before-boot.html) 即可ssh登录


### 初始化系统

更改主机名，修改密码以及连接Wi-Fi等工作都可以通过 raspi-config 命令来完成。

* k8s-master-1
* k8s-worker-1
* k8s-worker-2

### Master节点设置

#### 设置静态IP

```
$ cat << EOF >> /etc/dhcpcd.conf
interface wlan0
static ip_address=192.168.0.100/24
static routers=192.168.0.1
static domain_name_servers=8.8.8.8
EOF
```

其他机器相同设置，比如101, 102, 103，上面的设置是基于wifi连接，如果你的树莓派是通过网线连接，则需要把 `interface wlan0` 换成 `profile eth0`

#### 设置网络代理

这一步非常重要，因为树莓派是armhfp架构的，国内源对这个架构支持不好，可能安装的Docker不能在上面工作，因此需要设置网络代理，通过代理下载软件。

```
$ export http_proxy="http://192.168.3.21:7890"
$ export https_proxy="http://192.168.3.21:7890"
```

#### 安装docker
这个步骤依赖于上面的网络代理设置的，如果设置成功了，那么这步才可能成功执行。

```
$ curl -sSL get.docker.com | sh && sudo usermod pi -aG docker
$ newgrp docker
```

#### 关闭swap

在k8s 1.7以后，如果swap开启会报错

```
$ sudo dphys-swapfile swapoff && \
  sudo dphys-swapfile uninstall && \
  sudo update-rc.d dphys-swapfile remove
```
 
 官方系统是基于Debian的，需要运行：
 
 ```
 sudo systemctl disable dphys-swapfile
 ```

为了检测是否成功关闭，可以执行下面的命令，如果成功执行了，那么下面的命令将不会有任何的输出。

```
$ sudo swapon --summary
```

#### 编辑 /boot/cmdline.txt
打开 /boot/cmdline.txt，并在末尾添加如下指令。

```
cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
```

> 该步骤执行完成后，一定要重启！否则后面会有错误。

#### 安装kubernetes

该步骤也同样依赖于设置网络代理那一步骤！

为了保持整个集群的k8s版本统一，这里指定安装版本，我安装的是1.18.3，你们可以根据自己的情况安装，因为k8s版本和docker版本是有对应关系的

```
$ sudo su # 切换到root用户
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - && \
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list && \
apt-get update -q && \
apt-get install -qy kubeadm=1.18.3-00 kubectl=1.18.3-00 kubelet=1.18.3-00
```

> 切到root用户后，需要检查网络代理是否正常配置。

k8s安装完成后，可以将命令补全工具加入~/.bashrc等文件中

```
source <(kubectl completion bash)
source <(kubeadm completion bash)
```

#### 修改docker的网络代理
这一步非常重要！因为我们可能需要pull很多国内不能访问的镜像，因此需要设置docker网络代理，因为默认情况下，docker服务不是通过systemctl管理的，因此需要创建一个系统服务，具体的命令如下：

```
$ mkdir -pv /etc/systemd/system/docker.service.d
$ cat << EOF >> /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://172.16.3.12:1080/" "HTTPS_PROXY=http://172.16.3.12:1080/"
EOF 
$ systemctl daemon-reload
$ systemctl restart docker
```

> 参考：https://docs.docker.com/config/daemon/systemd/

#### 预先pull镜像

```
$ kubeadm config images pull -v3 --kubernetes-version=v1.18.3 --v=6
```

#### 初始化master节点

```
$ kubeadm init \ 
	--apiserver-advertise-address=192.168.3.23 \
	--kubernetes-version=v1.18.3 \
	--pod-network-cidr=10.244.0.0/16
```

初始化命令说明：

```
--apiserver-advertise-address
```

指明用 Master 的哪个 interface 与 Cluster 的其他节点通信。如果 Master 有多个 interface，建议明确指定，如果不指定，kubeadm 会自动选择有默认网关的 interface。

```
--pod-network-cidr
```

指定 Pod 网络的范围。Kubernetes 支持多种网络方案，而且不同网络方案对 --pod-network-cidr 有自己的要求，这里设置为 10.244.0.0/16 是因为我们将使用 flannel 网络方案，必须设置成这个 CIDR。

```
--kubernetes-version=v1.18.3 
```

关闭版本探测，因为它的默认值是stable-1，会导致从https://dl.k8s.io/release/stable-1.txt下载最新的版本号，我们可以将其指定为固定版本（最新版：v1.18.3）来跳过网络请求。

这步如果成功的话，会打印下面的消息：

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.16.3.1:6443 --token xo78oj.02cia85vdh285aqj --discovery-token-ca-cert-hash sha256:9517c72036f8261ac912adaf8339b65583fdaa7dbb8dd60054c6e84e8880a3fd
```
  
然后根据输出的提示，执行下面的语句：

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 上面的语句需要在pi用户下执行！否则不管用！

需要这些配置命令的原因是：Kubernetes 集群默认需要加密方式访问。所以，这几条命令，就是将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的.kube 目录下，kubectl 默认会使用这个目录下的授权信息访问 Kubernetes 集群。
如果不这么做的话，我们每次都需要通过 export KUBECONFIG 环境变量告诉 kubectl 这个安全配置文件的位置。
配置完成后centos用户就可以使用 kubectl 命令管理集群了。

### 验证

确认各个组件都处于healthy状态，查看节点状态

```
[pi@k8s-master-1 ~]$ kubectl get nodes 
NAME            STATUS     ROLES    AGE   VERSION
k8s-master-1   NotReady   master   36m   v1.18.3
```

你会发现master节点NotReady的消息，这是因为我们的网络需要依赖于flannel网络组件  

要让 Kubernetes Cluster 能够工作，必须安装 Pod 网络，否则 Pod 之间无法通信。
Kubernetes 支持多种网络方案，这里我们使用 flannel
执行如下命令部署 flannel：

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

然后在每个节点包括Master都执行以下命令:

```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

部署完成后，因为Pod创建需要稍微等待几秒钟，我们可以通过 kubectl get 重新检查 Pod 的状态：

```
pi@k8s-worker-1:~ $ kubectl get pod -n kube-system -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
coredns-66bff467f8-4vfmt               1/1     Running   0          14h   10.244.0.3     k8s-master-1   <none>           <none>
coredns-66bff467f8-6ln2g               1/1     Running   0          14h   10.244.0.2     k8s-master-1   <none>           <none>
etcd-k8s-master-1                      1/1     Running   0          14h   192.168.3.23   k8s-master-1   <none>           <none>
kube-apiserver-k8s-master-1            1/1     Running   0          14h   192.168.3.23   k8s-master-1   <none>           <none>
kube-controller-manager-k8s-master-1   1/1     Running   2          14h   192.168.3.23   k8s-master-1   <none>           <none>
kube-flannel-ds-arm-lc28w              1/1     Running   0          14h   192.168.3.23   k8s-master-1   <none>           <none>
kube-flannel-ds-arm-nq7hq              1/1     Running   2          10h   192.168.3.25   k8s-worker-2   <none>           <none>
kube-proxy-946hr                       1/1     Running   0          14h   192.168.3.23   k8s-master-1   <none>           <none>
kube-proxy-vnx7l                       1/1     Running   0          11h   192.168.3.24   k8s-worker-1   <none>           <none>
kube-proxy-zt8bl                       1/1     Running   0          10h   192.168.3.25   k8s-worker-2   <none>           <none>
kube-scheduler-k8s-master-1            1/1     Running   2          14h   192.168.3.23   k8s-master-1   <none>           <none>
```

```
[pi@k8s-master-1 ~]$ kubectl get nodes 
NAME          STATUS     ROLES    AGE   VERSION
k8s-master-1   Ready   master   36m   v1.18.3
```

至此，Kubernetes 的 Master 节点就部署完成了。如果你只需要一个单节点的 Kubernetes，现在你就可以使用了。不过，在默认情况下，Kubernetes 的 Master 节点是不能运行用户 Pod 的。

### 部署Worker节点

Kubernetes 的 Worker 节点跟 Master 节点几乎是相同的，它们运行着的都是一个 kubelet 组件。唯一的区别在于，在 kubeadm init 的过程中，kubelet 启动后，Master 节点上还会自动运行 kube-apiserver、kube-scheduler、kube-controller-manger 这三个系统 Pod。

节点机器只要k8s安装完成既可以，会在join的时候拉取master上的镜像，我们在 k8s-worker-1 和 k8s-worker-1 上分别执行如下命令，将其注册到 Cluster 中：

```
#执行以下命令将节点接入集群
kubeadm join 192.168.92.56:6443 --token 67kq55.8hxoga556caxty7s --discovery-token-ca-cert-hash sha256:7d50e704bbfe69661e37c5f3ad13b1b88032b6b2b703ebd4899e259477b5be69

#如果执行kubeadm init时没有记录下加入集群的命令，可以通过以下命令重新创建
kubeadm token create --print-join-command
```

然后根据提示，我们可以通过 kubectl get nodes 查看节点的状态：

```
pi@k8s-master-1:~ $ kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master-1   Ready    master   14h   v1.18.3
k8s-worker-1   Ready    <none>   11h   v1.18.3
k8s-worker-2   Ready    <none>   10h   v1.18.3
```

至此k8s集群搭建完成，更详细的问题查找和集群验证，可以查看上面centos的文章