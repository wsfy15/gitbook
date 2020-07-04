# kubeadm安装K8S

使用[kubeadm](https://k8smeetup.github.io/docs/reference/setup-tools/kubeadm) 一键部署工具安装kubernetes，准备两台虚拟机(4G内存)，一台作为master 节点，另一台为worker 节点。本次部署不包括高可用、授权、多租户、灾难备份等生产级别集群的功能，这些功能也可以通过kubeadm [实现](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/)。

## 安装

### 安装docker和kubeadm

```
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

$ cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y docker.io kubelet kubeadm kubectl
$ apt-mark hold kubelet kubeadm kubectl # 这几个包

$ 设置docker daemon
$ cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

$ swapoff -a

# 命令补全
$ source /etc/bash_completion
$ source <(kubectl completion bash)
$ source <(kubeadm completion bash)
```



### 部署master节点

编写`kubeadm init`的配置文件：

```
$ cat << EOF > kubeadm.yml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.17.8
controllerManager:
  extraArgs:
    horizontal-pod-autoscaler-sync-period: "10s"
    node-monitor-grace-period: "10s"
networking:
  podSubnet: 10.244.0.0/16
apiServer:
  extraArgs:
    advertise-address: 192.168.50.23
    runtime-config: "api/all=true"
EOF
```

> 指定`podSubnet`相当于`--pod-network-cidr`，这个值必须与后续安装的网络插件flannel配置文件里面的`Network`相同。

初始化master 节点：

```
$ unset HTTP_PROXY;unset HTTPS_PROXY;unset https_proxy;unset http_proxy
$ kubeadm init --config kubeadm.yml
......
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.50.23:6443 --token 4pm43h.968j6bt0jjfvki8n \
    --discovery-token-ca-cert-hash sha256:542d93b6a71118a769185a744b4e60abd641b26836f3cbd1197c770d7ecf8d8c
```

> 一旦想要重新安装或者配置文件有修改，最好先`kubeadm reset`，然后把etcd的存储以及k8s相关的配置给删了，如果没有删除配置文件目录，由于配置文件已存在，可能导致新修改的参数没有写入到相关文件中。这些操作可以通过下面的 restart 脚本自动化完成。
>
> ```
> # 需要手动写入，不然$(docker ps -aq)会被执行
> $ cat << EOF > restart.sh
> #!/bin/bash
> service docker restart
> docker rm -f $(docker ps -aq)
> service kubelet stop
> rm -r /etc/kubernetes/ /var/lib/kubelet /var/run/kubernetes /var/lib/cni /var/lib/etcd
> EOF
> ```
>
> 

输出的最后几行，就是用于在worker 节点上执行，加入该集群的命令。kubeadm 提示第一次使用 Kubernetes 集群所需要的配置命令：

```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Kubernetes 集群默认需要加密方式访问。所以，上面这几条命令，就是将刚刚部署生成的 Kubernetes 集群的安全配置文件，保存到当前用户的`.kube` 目录下，`kubectl `默认会使用这个目录下的授权信息访问 Kubernetes 集群。如果不这么做的话，那就每次都需要通过` export KUBECONFIG` 环境变量告诉 `kubectl `这个安全配置文件的位置。如果都不做，直接执行`kubectl`命令，就会报错`The connection to the server localhost:8080 was refused - did you specify the right host or port?`。

使用 `kubectl get `命令来查看当前唯一一个节点的状态：

```
$ kubectl get nodes
NAME      STATUS     ROLES    AGE     VERSION
manager   NotReady   master   8m48s   v1.17.3
```

`get `指令输出的结果里，Master 节点的状态是 NotReady，这时候，就可以通过`kubectl describe`指令查看这个节点对象的详细信息、状态和事件了：

```
 $ kubectl describe node manager
 Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 30 Jun 2020 13:10:09 +0000   Tue, 30 Jun 2020 12:25:01 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 30 Jun 2020 13:10:09 +0000   Tue, 30 Jun 2020 12:25:01 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 30 Jun 2020 13:10:09 +0000   Tue, 30 Jun 2020 12:25:01 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Tue, 30 Jun 2020 13:10:09 +0000   Tue, 30 Jun 2020 12:25:01 +0000   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

NotReady 的原因是`network plugin is not ready`，即未部署任何网络插件。此时检查这个节点上各个系统pod 的状态：

```
$ kubectl get pods -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-6955765f44-4lqlv          0/1     Pending   0          20s
coredns-6955765f44-gqkjc          0/1     Pending   0          20s
etcd-manager                      1/1     Running   0          25s
kube-apiserver-manager            1/1     Running   0          25s
kube-controller-manager-manager   1/1     Running   2          25s
kube-proxy-4qj7d                  1/1     Running   0          20s
kube-scheduler-manager            1/1     Running   0          25s
```

> kube-system 是Kubernetes 预留的系统 Pod 的工作空间（Namepsace，注意它并不是 Linux Namespace，
> 它只是 Kubernetes 划分不同工作空间的单位）。
>
> 可以通过`kubectl get namespace`查看所有的namespace。

coredns 这个依赖网络的Pod 处于Pending状态，即调度失败。



### 安装网络插件

在 Kubernetes “一切皆容器”的设计理念指导下，部署网络插件非常简单，以部署flannel插件为例，只需要先到[github](https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml)下载yml文件，然后执行`kubectl apply` 指令：

```
$ kubectl apply -f kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created

$ kubectl get pods -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-6955765f44-4lqlv          1/1     Running   0          33s
coredns-6955765f44-gqkjc          1/1     Running   0          33s
etcd-manager                      1/1     Running   0          26s
kube-apiserver-manager            1/1     Running   0          26s
kube-controller-manager-manager   1/1     Running   0          26s
kube-flannel-ds-amd64-5pwqq       1/1     Running   0          26s
kube-proxy-4qj7d                  1/1     Running   0          33s
kube-scheduler-manager            1/1     Running   0          26s
```

所有的系统Pod 都启动成功了，Flannel 插件在`kube-system`下面新建了 `kube-flannel-ds-amd64-5pwqq` Pod，一般来说，这些 Pod 就是容器网络插件在**每个节点上**的控制组件。

> Kubernetes 支持容器网络插件，是因为使用 CNI 的通用接口，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过 CNI 接入 Kubernetes，比如 Flannel、Calico、Canal、Romana 等等，它们的部署方式也都是类似的“一键部署”。



### Taint/Toleration

到这里，Kubernetes 的 Master 节点就部署完成了。如果只需要一个单节点的 Kubernetes，现在就可以使用了。不过，在默认情况下，Kubernetes 的 Master 节点是不能运行**用户 Pod** 的，所以需要调整 Master 执行 Pod 的策略。

这个策略依靠的是 Kubernetes 的 Taint/Toleration 机制。

一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。

除非，有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。

为节点打上“污点”（Taint）的命令是：

```
$ kubectl taint nodes node1 foo=bar:NoSchedule
```

node1 节点上就会增加一个键值对格式的 Taint，即：`foo=bar:NoSchedule`。其中`:`后面的 `NoSchedule`，意味着这个 Taint 只会在调度新 Pod 时产生作用，而不会影响已经在 node1上运行的 Pod，哪怕它们没有 Toleration。



为Pod 声明 Toleration 的方法是在Pod 的yml 文件中的 `spec`部分，加入`tolerations`字段：

```
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "foo"
	operator: "Equal"
	value: "bar"
	effect: "NoSchedule"
```

这个 Toleration 的含义是，这个 Pod 能“容忍”所有键值对为 `foo=bar` 的 Taint。



那么要调整master 节点执行 Pod 的策略，首先看它被打上了什么Taint：

```
$ kubectl describe node manager
Name:               manager
Roles:              master
......
Taints:             node-role.kubernetes.io/master:NoSchedule
```

Master 节点默认被加上了`node-role.kubernetes.io/master:NoSchedule`这样一个“污点”，其中“键”是`node-role.kubernetes.io/master`，而没有提供“值”。为了容忍这个”污点“，可以使用`Exists` operator：

```
apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

除了容忍”污点“，还可以把”污点“删除：

```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```

注意到`node-role.kubernetes.io/master`键后面加上了一个短横线`-`，这个格式就意味着移除所有以`node-role.kubernetes.io/master`为键的Taint。



### 部署worker节点

```
$ kubeadm join 192.168.50.23:6443 --token 4pm43h.968j6bt0jjfvki8n \
    --discovery-token-ca-cert-hash sha256:542d93b6a71118a769185a744b4e60abd641b26836f3cbd1197c770d7ecf8d8c
......
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

# 过一会执行
$  docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
40b64fa47d41        4e9f801d2217           "/opt/bin/flanneld -…"   12 seconds ago      Up 11 seconds
        k8s_kube-flannel_kube-flannel-ds-amd64-xv5s7_kube-system_dfc1be18-63b7-4c0c-8318-d4edb2a0c804_0
18b53697a5c8        157d95474c77           "/usr/local/bin/kube…"   55 seconds ago      Up 54 seconds
        k8s_kube-proxy_kube-proxy-kwz6c_kube-system_84f7582b-9713-41af-ba13-15d20124541e_0
7a876d08a300        k8s.gcr.io/pause:3.1   "/pause"                 56 seconds ago      Up 55 seconds
        k8s_POD_kube-flannel-ds-amd64-xv5s7_kube-system_dfc1be18-63b7-4c0c-8318-d4edb2a0c804_0
badaf5bb9d3d        k8s.gcr.io/pause:3.1   "/pause"                 57 seconds ago      Up 55 seconds
        k8s_POD_kube-proxy-kwz6c_kube-system_84f7582b-9713-41af-ba13-15d20124541e_0
        
$ kubectl get nodes
NAME      STATUS     ROLES    AGE    VERSION
client    Ready      <none>   50s    v1.17.3
manager   Ready      master   7m8s   v1.17.3
```



执行` kubeadm join`的时候如果报错如下，可能是worker节点的`kubelet`版本比master节点的`kubernetes`版本高。

```
error execution phase kubelet-start: cannot get Node "client": nodes "client" is forbidden: User "system:bootstrap:5njads" cannot get resource "nodes" in API group "" at the cluster scope
```



### 安装Dashboard 插件

[Dashboard](https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/)为用户提供一个可视化的Web 界面来查看当前集群的各种信息，它的安装也很简单，只需要执行：

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

$ kubectl get pods -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-64698f5d56-hq7n8   1/1     Running   0          11m
kubernetes-dashboard-567b64d68b-pqgb6        1/1     Running   0          11m
```

如果不想部署在master 节点上，可以注释yaml 文件中`tolerations`的相关部分。

提前在worker 节点上`docker pull`需要的image，如果遇到网络不通，可以在其他网络状况良好的机器上先pull image，然后`docker save ${imageName} > ${imageName}.tar.gz`，然后把压缩包通过`scp`拷贝到worker 节点上。



### 安装容器存储插件

很多时候我们需要用数据卷（Volume）把宿主机上的目录或者文件挂载进容器的 Mount Namespace 中，从而达到容器和宿主机共享这些目录或者文件的目的。容器里的应用，也就可以在这些数据卷中新建和写入文件。

可是，随便在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。这是容器最典型的特征之一：**无状态**。

而容器的持久化存储，就是用来保存容器存储状态的重要手段：**存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。**这样，无论在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。这就是“**持久化**”的含义。

由于 Kubernetes 本身的松耦合设计，绝大多数存储项目，比如 Ceph、GlusterFS、NFS 等，都可以为 Kubernetes 提供持久化存储能力。以Rook 为例，它是一个基于 Ceph 的 Kubernetes 存储插件（也支持EdgeFS等存储实现）。不过，不同于对 Ceph 的简单封装，Rook 在自己的实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得这个项目变成了一个完整的、生产级别可用的容器存储插件。

根据[rook的文档](https://rook.github.io/docs/rook/v1.3/ceph-quickstart.html)，部署rook只需要依次apply 下面三个yaml 文件：

```
# 因为rook/ceph容器需要运行在三个不同的机器上，而实验环境只有一个master和两个worker，所以需要让master节点也能运行用户pod
$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/manager untainted

# 部署 operator
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.3/cluster/examples/kubernetes/ceph/common.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.3/cluster/examples/kubernetes/ceph/operator.yaml

$ kubectl get po -n rook-ceph -o wide
NAME                                 READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE   READINESS GATES
NAME                                 READY   STATUS    RESTARTS   AGE     IP           NODE      NOMINATED NODE   READINESS GATES
rook-ceph-operator-5b6674cb6-55m69              1/1     Running   0          78s   10.244.1.13   client2   <none>           <none>
rook-discover-4kwtj                             1/1     Running   0          65s   10.244.2.13   client3   <none>           <none>
rook-discover-96rnd                             1/1     Running   0          65s   10.244.0.10   manager   <none>           <none>
rook-discover-t55z8                             1/1     Running   0          65s   10.244.1.14   client2   <none>           <none>


# 创建 rook ceph 集群
$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/release-1.3/cluster/examples/kubernetes/ceph/cluster.yaml


$ kubectl -n rook-ceph get pod -o wide
csi-cephfsplugin-lhpwd                          3/3     Running   0          52s   10.0.2.15     client3   <none>           <none>
csi-cephfsplugin-lxs9j                          3/3     Running   0          52s   10.0.2.15     client2   <none>           <none>
csi-cephfsplugin-ppr9j                          3/3     Running   0          52s   10.0.2.15     manager   <none>           <none>
csi-cephfsplugin-provisioner-7469b99d4b-n2qqg   5/5     Running   0          52s   10.244.0.11   manager   <none>           <none>
csi-cephfsplugin-provisioner-7469b99d4b-vjdtn   5/5     Running   0          52s   10.244.2.16   client3   <none>           <none>
csi-rbdplugin-hjg56                             3/3     Running   0          53s   10.0.2.15     client2   <none>           <none>
csi-rbdplugin-lfffz                             3/3     Running   0          53s   10.0.2.15     manager   <none>           <none>
csi-rbdplugin-mgbmb                             3/3     Running   0          53s   10.0.2.15     client3   <none>           <none>
csi-rbdplugin-provisioner-865f4d8d-4rnbg        6/6     Running   0          53s   10.244.1.15   client2   <none>           <none>
csi-rbdplugin-provisioner-865f4d8d-r6frx        6/6     Running   0          53s   10.244.2.17   client3   <none>           <none>
rook-ceph-mon-c-canary-5d59fc6588-9xwl6         1/1     Running   0          26s   10.244.0.12   manager   <none>           <none>
rook-ceph-operator-5b6674cb6-55m69              1/1     Running   0          78s   10.244.1.13   client2   <none>           <none>
rook-discover-4kwtj                             1/1     Running   0          65s   10.244.2.13   client3   <none>           <none>
rook-discover-96rnd                             1/1     Running   0          65s   10.244.0.10   manager   <none>           <none>
rook-discover-t55z8                             1/1     Running   0          65s   10.244.1.14   client2   <none>           <none>
```

这样，一个基于 Rook 持久化存储集群就以容器的方式运行起来了，而接下来在 Kubernetes 上创建的所有 Pod 就能够通过 Persistent Volume（PV）和 Persistent Volume Claim（PVC）的方式，在容器里挂载由 Ceph 提供的数据卷了。而 Rook 则会负责这些数据卷的生命周期管理、灾难备份等运维工作。





## 原理

### 容器化部署？

给每个 Kubernetes 组件做一个容器镜像，然后在每台宿主机上用 `docker run` 指令启动这些组件容器，部署不就完成了吗？

但这会带来一个问题：**如何容器化 kubelet**。

 kubelet 是 Kubernetes 用来操作 Docker 等容器运行时的核心组件。可是，除了跟容器运行时打交道外，kubelet 在配置容器网络、管理容器数据卷时，都需要**直接操作宿主机**。

如果现在 kubelet 运行在一个容器里，那么直接操作宿主机就会变得很麻烦。对于网络配置来说还好，kubelet 容器可以通过不开启 Network Namespace（即 Docker 的 host network模式）的方式，直接共享宿主机的网络栈。

可是，要让 kubelet 隔着容器的 Mount Namespace 和文件系统，操作宿主机的文件系统，就有点儿困难了。比如，如果用户想要使用 NFS 做容器的持久化数据卷，那么 kubelet 就需要在容器进行绑定挂载前，在宿主机的指定目录上，先挂载 NFS 的远程目录。

但由于现在 kubelet 是运行在容器里的，这就意味着它执行的`mount -F nfs`命令，被隔离在一个单独的 Mount Namespace 中。即，kubelet 做的挂载操作，不能被“传播”到宿主机上。

虽然可以使用 `setns()` 系统调用，在宿主机的 Mount Namespace 中执行这些挂载操作；或者让 Docker 支持一个`–mnt=host` 的参数。但是，到目前为止，在容器里运行 kubelet，依然没有很好的解决办法，所以不推荐用容器去部
署 Kubernetes 项目。

因此，kubeadm的做法是：**把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。**



### kubeadm init

执行 `kubeadm init `指令后，**kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes。**这一步检查，称为“Preflight Checks”，它可以为你省掉很多后续的麻烦。Preflight Checks 包括了很多方面，比如：

- Linux 内核的版本必须是否是 3.10 以上？
- Linux Cgroups 模块是否可用？
- 机器的 hostname 是否标准？在 Kubernetes 项目里，机器的名字以及一切存储在 Etcd 中的API 对象，都必须使用标准的 DNS 命名（RFC 1123）。
- 用户安装的 kubeadm 和 kubelet 的版本是否匹配？
- 机器上是不是已经安装了 Kubernetes 的二进制文件？
- Kubernetes 的工作端口 10250/10251/10252 端口是不是已经被占用？
- `ip、mount` 等 Linux 指令是否存在？
- Docker 是否已经安装并且运行？
- ……



**在通过了 Preflight Checks 之后，kubeadm 就要开始生成 Kubernetes 对外提供服务所需的各种证书和对应的目录。**

Kubernetes 对外提供服务时，除非开启“不安全模式”，否则都要通过 HTTPS 才能访问kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。

kubeadm 为 Kubernetes 生成的证书文件都放在 Master 节点的` /etc/kubernetes/pki` 目录下。在这个目录下，最主要的证书文件是 `ca.crt `和对应的私钥` ca.key`。

此外，用户使用 kubectl 获取容器日志等 streaming 操作时，需要通过 kube-apiserver 向kubelet 发起请求，这个连接也必须是安全的。kubeadm 为这一步生成的是 `apiserver-kubeletclient.crt` 文件，对应的私钥是 `apiserver-kubelet-client.key`。

除此之外，Kubernetes 集群中还有 Aggregate APIServer 等特性，也需要用到专门的证书。

如果指定目录里都已存在证书文件，就会跳过这一步。



**证书生成后，kubeadm 接下来会为其他组件生成访问 kube-apiserver 所需的配置文件。**这些文件的路径是：`/etc/kubernetes/xxx.conf`：

```
$ ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  manifests  pki  scheduler.conf
```

这些`.conf`文件里面记录的是，当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如 scheduler，kubelet 等），可以直接加载相应的文件，使用里面的信息与kube-apiserver 建立安全连接。



**接下来，kubeadm 会为 Master 组件生成 Pod 配置文件。**Kubernetes 有三个在 Master 节点上运行的组件：kube-apiserver、kube-controller-manager、kubescheduler，它们都会以 Pod 的方式部署。但这时，Kubernetes 集群尚不存在，难道 kubeadm 会直接执行 `docker run` 来启动这些容器吗？

当然不是。**在 Kubernetes 中，有一种特殊的容器启动方法叫做 Static Pod**。它允许你把要部署的 Pod 的 YAML 文件放在一个指定的目录里。这样，**当这台机器上的 kubelet 启动时，它会自动检查这个目录，加载所有的 Pod YAML 文件，然后在这台机器上启动它们。**

从这一点也可以看出，kubelet 在 Kubernetes 项目中的地位非常高，在设计上它就是一个完全独立的组件，而其他 Master 组件，则更像是辅助性的系统容器。在 kubeadm 中，Master 组件的 YAML 文件会被生成在`/etc/kubernetes/manifests` 路径下：

```
$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

可以通过修改这些yaml 文件修改一个已有集群的配置，这些yaml 文件里的参数也可以在部署时指定。

一旦这些 YAML 文件出现在被 kubelet 监视的 `/etc/kubernetes/manifests` 目录下，kubelet 就会自动创建这些 YAML 文件中定义的 Pod，即 Master 组件的容器。Master 容器启动后，kubeadm 会通过检查 `localhost:6443/healthz` 这个 Master 组件的健康检查URL，等待 Master 组件完全运行起来。



**然后，kubeadm 就会为集群生成一个 bootstrap token。**这个 token 可以让其他节点通过 kubeadm join 加入到这个集群当中，前提是这些节点已经安装了 kubelet 和 kubadm 。这个 token 的值和使用方法会，会在 `kubeadm init` 结束后被打印出来。



**在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 ConfigMap 的方式保存在 Etcd 当中，供后续部署 Node 节点使用。**这个 ConfigMap 的名字是 cluster-info。



**kubeadm init 的最后一步，就是安装默认插件。**Kubernetes 默认 kube-proxy 和 DNS 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。其实，这两个插件也只是两个容器镜像而已，所以 kubeadm 只要用 Kubernetes 客户端创建两个 Pod 就可以了。



### kubeadm join

`kubeadm init` 生成 bootstrap token 之后，就可以在任意一台安装了kubelet 和 kubeadm 的机器上执行 `kubeadm join`了。为什么执行 `kubeadm join` 需要这样一个 token 呢？

因为，任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册。可是，要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（CA 文件）。可是，为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。

所以，kubeadm 至少需要发起一次“不安全模式”的访问到 kube-apiserver，从而拿到保存在ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。而 bootstrap token，扮演的就是这个过程中的安全验证的角色。

只要有了 cluster-info 里的 kube-apiserver 的地址、端口、证书，kubelet 就可以以“安全模式”连接到 apiserver 上，这样一个新的节点就部署完成了。



### 配置 kubeadm 的部署参数

为了定制集群组件参数，可以在`kubeadm init`的时候，传入配置文件：

```
$ kubeadm init --config kubeadm.yaml
```

这个yaml 文件支持的参数可以参考[官方文档](https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2)，之后kubeadm 就会使用yaml 文件里指定的信息替换` /etc/kubernetes/manifests`目录中相应的yaml 文件里相关字段的信息了。

这个文件支持下列配置类型：

- `InitConfiguration`：与bootstrapTokens、nodeRegistration相关
- `ClusterConfiguration`：与集群的网络、etcd、apiserver相关
- `KubeletConfiguration`：kubelet 的配置
- `KubeProxyConfiguration`：kube-proxy 的配置
- `JoinConfiguration`：`kubeadm join`时可以使用的配置，包含NodeRegistration、APIEndpoint等

下面是官方提供的一份示例配置文件：

```
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
bootstrapTokens:
- token: "9a08jv.c0izixklcxtmnze7"
  description: "kubeadm bootstrap token"
  ttl: "24h"
- token: "783bde.3f89s0fje9f38fhf"
  description: "another bootstrap token"
  usages:
  - authentication
  - signing
  groups:
  - system:bootstrappers:kubeadm:default-node-token
nodeRegistration:
  name: "ec2-10-100-0-1"
  criSocket: "/var/run/dockershim.sock"
  taints:
  - key: "kubeadmNode"
    value: "master"
    effect: "NoSchedule"
  kubeletExtraArgs:
    cgroup-driver: "cgroupfs"
  ignorePreflightErrors:
  - IsPrivilegedUser
localAPIEndpoint:
  advertiseAddress: "10.100.0.1"
  bindPort: 6443
certificateKey: "e6a2eb8581237ab72a4f494f30285ec12a9694d750b9785706a83bfcbbbd2204"
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
etcd:
  # one of local or external
  local:
    imageRepository: "k8s.gcr.io"
    imageTag: "3.2.24"
    dataDir: "/var/lib/etcd"
    extraArgs:
      listen-client-urls: "http://10.100.0.1:2379"
    serverCertSANs:
    -  "ec2-10-100-0-1.compute-1.amazonaws.com"
    peerCertSANs:
    - "10.100.0.1"
  # external:
    # endpoints:
    # - "10.100.0.1:2379"
    # - "10.100.0.2:2379"
    # caFile: "/etcd/kubernetes/pki/etcd/etcd-ca.crt"
    # certFile: "/etcd/kubernetes/pki/etcd/etcd.crt"
    # keyFile: "/etcd/kubernetes/pki/etcd/etcd.key"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.100.0.1/24"
  dnsDomain: "cluster.local"
kubernetesVersion: "v1.12.0"
controlPlaneEndpoint: "10.100.0.1:6443"
apiServer:
  extraArgs:
    authorization-mode: "Node,RBAC"
  extraVolumes:
  - name: "some-volume"
    hostPath: "/etc/some-path"
    mountPath: "/etc/some-pod-path"
    readOnly: false
    pathType: File
  certSANs:
  - "10.100.1.1"
  - "ec2-10-100-0-1.compute-1.amazonaws.com"
  timeoutForControlPlane: 4m0s
controllerManager:
  extraArgs:
    "node-cidr-mask-size": "20"
  extraVolumes:
  - name: "some-volume"
    hostPath: "/etc/some-path"
    mountPath: "/etc/some-pod-path"
    readOnly: false
    pathType: File
scheduler:
  extraArgs:
    address: "10.100.0.1"
  extraVolumes:
  - name: "some-volume"
    hostPath: "/etc/some-path"
    mountPath: "/etc/some-pod-path"
    readOnly: false
    pathType: File
certificatesDir: "/etc/kubernetes/pki"
imageRepository: "k8s.gcr.io"
useHyperKubeImage: false
clusterName: "example-cluster"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# kubelet specific options here
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
# kube-proxy specific options here
```



 kubeadm 的源代码，直接就在` kubernetes/cmd/kubeadm` 目录下，是 Kubernetes 项目的一部分。其中，`app/phases` 文件夹下的代码，对应的就是每一个具体步骤。



### 不足

这里的部署不能用于生产环境，因为master、Etcd都只有一个节点，保证不了高可用。不过官方也提供了[高可用部署的方法](https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/)。也可以用SaltStack、[kops](https://github.com/kubernetes/kops)、[kubeasz](https://github.com/easzlab/kubeasz) 等工具进行部署。





## 宿主机reboot后不能正常启动

```
$ journalctl -u kubelet -f
Jul 02 05:18:27 manager kubelet[6500]: E0702 05:18:27.639062    6500 reflector.go:153] k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:46: Failed to list *v1.Pod: Get https://192.168.50.23:6443/api/v1/pods?fieldSelector=spec.nodeName%3Dmanager&limit=500&resourceVersion=0: dial tcp 192.168.50.23:6443: connect: connection refused

$ docker ps
# no container
```

这时执行下面的命令，再执行`docker ps`也可以发现容器启动了。

```
$ swapoff -a
$ systemctl restart kubelet.service
```

