# Service

Kubernetes 之所以需要 Service，一方面是因为 Pod 的 IP 不是固定的，另一方面则是因为一组 Pod 实例之间总会有负载均衡的需求。

## 使用

一个最典型的 Service 定义，如下所示：

```
apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
```

使用了 selector 字段来声明这个 Service 只代理携带了 `app=hostnames` 标签的 Pod。并且，这个 Service 的 80 端口，代理的是 Pod 的9376 端口。



定义一个Deployment：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
```

这个应用的作用，就是每次访问 9376 端口时，返回它自己的 hostname。



被 selector 选中的 Pod，就称为 Service 的 **Endpoints**，可以使用 `kubectl get ep` 命令看到它们：

```
$ kubectl get ep hostnames
NAME        ENDPOINTS                                         AGE
hostnames   10.244.0.2:9376,10.244.1.2:9376,10.244.2.3:9376   2m1s
```

> 只有处于 Running 状态，且 `readinessProbe `检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。



通过该 Service 的 VIP 就可以访问到它所代理的 Pod ：

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hostnames    ClusterIP   10.110.61.217   <none>        80/TCP     3m11s

$ kubectl get po -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP           NODE      NOMINATED NODE 
hostnames-755d96487f-6hhr9   1/1     Running   0          21m   10.244.0.6   manager   <none>         
hostnames-755d96487f-mx55n   1/1     Running   0          21m   10.244.1.5   client2   <none>         
hostnames-755d96487f-qtt26   1/1     Running   0          29m   10.244.2.5   client3   <none>         

$ curl 10.110.61.217
hostnames-755d96487f-6hhr9
$ curl 10.110.61.217
hostnames-755d96487f-qtt26
$ curl 10.110.61.217
hostnames-755d96487f-mx55n
```

这个 VIP 地址是 Kubernetes 自动为 Service 分配的。而像上面这样，通过三次连续不断地访问 Service 的 VIP 地址和代理端口 80，它就为我们依次返回了三个 Pod 的 hostname。这也正印证了 Service 提供的是 Round Robin 方式的负载均衡。对于这种方式，我们称为： ClusterIP 模式的 Service。



## 原理

Kubernetes 里的 Service 究竟是如何工作的呢？



### iptables

实际上，**Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。**

对于前面创建的 hostnames Service 来说，一旦它被提交给Kubernetes，那么 kube-proxy (运行在每个节点上)就可以通过 Service 的 Informer 感知到这样一个 Service 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则：

```
$ iptables-save
...
-A KUBE-SERVICES -d 10.110.61.217/32 -p tcp -m comment --comment "default/hostnames:default cluster IP" -m tcp --dport 80 -j KUBE-SVC-ODX2UBAZM7RQWOIU
```

这是往`KUBE-SERVICES`链追加一条规则：凡是目的地址是 10.110.61.217、目的端口是 80 的 IP包，都应该跳转到名叫`KUBE-SVC-ODX2UBAZM7RQWOIU`的iptables 链进行处理。

10.110.61.217 正是这个 Service 的 VIP，所以这一条规则，就为这个Service 设置了一个固定的入口地址。并且，由于 10.0.1.17510.110.61.217 只是一条 iptables 规则上的配置，并没有真正的网络设备，所以 **`ping `这个地址，是不会有任何响应的**。

`KUBE-SVC-ODX2UBAZM7RQWOIU` 链也是一组规则的集合：

```
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-2N4TWC53YU5D2546
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-LZSOAAIF344LOQYR
-A KUBE-SVC-ODX2UBAZM7RQWOIU -m comment --comment "default/hostnames:default" -j KUBE-SEP-3C5XT2CPGWXUSCZF
```

这是一组随机模式（`-–mode random`）的 iptables 链。由于 iptables 规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同，应该将它们的 `probability `字段的值分别设置为 1/3（0.333…）、1/2和 1。

这么设置的原理很简单：第一条规则被选中的概率就是 1/3；而如果第一条规则没有被选中，那么这时候就只剩下两条规则了，所以第二条规则的 `probability `就必须设置为 1/2；类似地，最后一条就必须设置为 1。

而这三条规则最后跳转到新的链，其实都是DNAT 规则，但在 DNAT 规则之前，iptables 对流入的 IP包还设置了一个“标志”（`-–set-xmark`）。

```
-A KUBE-SEP-2N4TWC53YU5D2546 -s 10.244.0.6/32 -m comment --comment "default/hostnames:default" -j KUBE-MARK-MASQ
-A KUBE-SEP-2N4TWC53YU5D2546 -p tcp -m comment --comment "default/hostnames:default" -m tcp -j DNAT --to-destination 10.244.0.6:9376

-A KUBE-SEP-LZSOAAIF344LOQYR -s 10.244.1.5/32 -m comment --comment "default/hostnames:default" -j KUBE-MARK-MASQ
-A KUBE-SEP-LZSOAAIF344LOQYR -p tcp -m comment --comment "default/hostnames:default" -m tcp -j DNAT --to-destination 10.244.1.5:9376

-A KUBE-SEP-3C5XT2CPGWXUSCZF -s 10.244.2.5/32 -m comment --comment "default/hostnames:default" -j KUBE-MARK-MASQ
-A KUBE-SEP-3C5XT2CPGWXUSCZF -p tcp -m comment --comment "default/hostnames:default" -m tcp -j DNAT --to-destination 10.244.2.5:9376

-A KUBE-MARK-MASQ -j MARK --set-xmark 0x4000/0x4000
```

DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成`-–to-destination` 所指定的新的目的地址和端口。这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

这样，访问 Service VIP 的 IP 包经过上述 iptables 处理之后，就已经变成了访问具体某一个后端 Pod 的 IP 包了。这些 Endpoints 对应的 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的。



### ipvs

kube-proxy 通过 iptables 处理 Service 的过程，需要在宿主机上设置相当多的 iptables 规则。而且，kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。如果宿主机上有大量的 Pod ，成百上千条 iptables 规则不断地被刷新，会大量占用该宿主机的 CPU 资源，甚至会让宿主机“卡”在这个过程中。所以说，一直以来，基于iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。

而 IPVS 模式的 Service，就是解决这个问题的一个行之有效的方法。IPVS 模式的工作原理，跟 iptables 模式类似。在创建 Service 之后，kube proxy 首先会**在宿主机上创建一个虚拟网卡**（`kube-ipvs0`），并**为它分配 Service VIP 作为 IP 地址**，如下所示：

```
# ip addr
 ...
 73：kube-ipvs0：<BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN qlen 1000
 link/ether 1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
 inet 10.0.1.175/32 scope global kube-ipvs0
 valid_lft forever preferred_lft forever
```

接下来，kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略。

```
# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
 Prot LocalAddress:Port Scheduler Flags
 -> RemoteAddress:Port Forward Weight ActiveConn InActConn
 TCP 10.102.128.4:80 rr
 -> 10.244.3.6:9376 Masq 1 0 0
 -> 10.244.1.7:9376 Masq 1 0 0
 -> 10.244.2.3:9376 Masq 1 0 0
```

这三个 IPVS 虚拟主机的 IP 地址和端口，对应的正是三个被代理的 Pod。这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了。

相比于 iptables，IPVS 在内核中的实现也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。



IPVS 模块只负责上述的**负载均衡和代理功能**。而一个完整的 Service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。不过，这些辅助性的iptables 规则数量有限，也不会随着 Pod 数量的增加而增加。所以，在大规模集群里，建议为 kube-proxy 设置`–proxy-mode=ipvs` 来开启这个功能。它为 Kubernetes 集群规模带来的提升，还是非常巨大的。



## DNS

在 Kubernetes 中，Service 和 Pod 都会被分配对应的 DNS A 记录（从域名解析 IP 的记录）。

对于 ClusterIP 模式的 Service 来说，它的 A 记录的格式是：`my-svc.my-namespace.svc.cluster.local`。当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

```
$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
/ # nslookup  hostnames.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames.default.svc.cluster.local
Address 1: 10.110.61.217 hostnames.default.svc.cluster.local
```



对于指定了`clusterIP=None` 的 Headless Service 来说，它的 A 记录的格式也是：`my-svc.my-namespace.svc.cluster.local`。但是，**在访问这条 A 记录的时候，它返回的是所有被代理的 Pod 的 IP 地址的集合**。当然，如果DNS 客户端没办法解析这个集合的话，它可能会只会拿到第一个Pod 的 IP 地址。



如果为 Pod 指定了 Headless Service，并且 Pod 本身声明了 `hostname `和 `subdomain`字段，那么这时候 Pod 的 A 记录就会变成：`${hostname}.${subdomain}.${namespace}.svc.cluster.local`。

```
apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo # Actually, no port is needed.
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox:1.28
    command:
      - sleep
      - "3600"
    name: busybox
```



> 在 Kubernetes 里，`/etc/hosts` 文件是单独挂载的，这也是为什么 kubelet 能够对 hostname 进行修改并且 Pod 重建后依然有效的原因。这跟 Docker 的 Init 层是一个原理。



## 对外提供服务

### NodePort

Service 的访问信息在 Kubernetes 集群之外，其实是无效的。因为所谓 Service 的访问入口，其实就是每台宿主机上由 kube-proxy 生成的iptables 规则，以及 kube-dns 生成的 DNS 记录。而一旦离开了这个集群，这些信息对用户来说，也就自然没有作用了。

所以，在使用 Kubernetes 的 Service 时，一个必须要面对和解决的问题就是：如何从外部（Kubernetes 集群之外），访问到 Kubernetes 里创建的 Service？最常用的一种方式就是NodePort：

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  type: NodePort
  ports:
  - port: 80 # VIP 的端口
    targetPort: 80 # Pod 的端口
    nodePort: 30007 # 宿主机 的端口
    protocol: TCP
    name: http
  - port: 443
    protocol: TCP
    name: https
  selector:
    run: my-nginx
```

这个 Service 定义了它的类型是 `type=NodePort`。然后，在 `ports `字段里声明了 Service 的 30007 端口代理 Pod 的 80 端口。

如果不显式地声明 `nodePort `字段，Kubernetes 就会自动分配随机的可用端口来设置代理。这个端口的范围默认是 30000-32767，可以通过 kube-apiserver 的`-–service-nodeport-range` 参数来修改它。

现在，要访问这个 Service，只需要访问：`<任意一台宿主机的IP>:8080` 就可以访问到某一个被代理的 Pod 的 80 端口了。

显然，kube-proxy 要做的，就是在每台宿主机上生成 iptables 规则：

```
-A KUBE-SERVICES -d 10.110.155.111/32 -p tcp -m comment --comment "default/my-nginx:http cluster IP" -m tcp --dport 9376 -j KUBE-SVC-SV7AMNAGZFKZEMQ4
-A KUBE-SERVICES -m comment --comment "kubernetes service nodeports; NOTE: this must be the last rule in this chain" -m addrtype --dst-type LOCAL -j KUBE-NODEPORTS

-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx:http" -m tcp --dport 30007 -j KUBE-SVC-SV7AMNAGZFKZEMQ4

-A KUBE-SVC-SV7AMNAGZFKZEMQ4 -m comment --comment "default/my-nginx:http" -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-Y6JQYAWWW54SDVHW
-A KUBE-SVC-SV7AMNAGZFKZEMQ4 -m comment --comment "default/my-nginx:http" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-YJPSRYALWYXINVEB
-A KUBE-SVC-SV7AMNAGZFKZEMQ4 -m comment --comment "default/my-nginx:http" -j KUBE-SEP-QDENARIFPFYV2MYI

-A KUBE-SEP-Y6JQYAWWW54SDVHW -p tcp -m comment --comment "default/my-nginx:http" -m tcp -j DNAT --to-destination 10.244.0.6:9376
-A KUBE-SEP-YJPSRYALWYXINVEB -p tcp -m comment --comment "default/my-nginx:http" -m tcp -j DNAT --to-destination 10.244.1.5:9376
-A KUBE-SEP-QDENARIFPFYV2MYI -p tcp -m comment --comment "default/my-nginx:http" -m tcp -j DNAT --to-destination 10.244.2.5:9376
```



在 NodePort 方式下，Kubernetes 会在 IP 包离开宿主机发往目的 Pod 时，对这个 IP 包做一次 SNAT 操作：

```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

这条规则设置在 POSTROUTING 检查点，也就是说，它给即将离开这台主机的 IP 包，进行了一次 SNAT 操作，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）。

当然，这个 SNAT 操作只需要对 Service 转发出来的 IP 包进行（否则普通的 IP 包就被影响了）。而 iptables 做这个判断的依据，就是查看该 IP 包是否有一个`0x4000`的“标志”，这个标志正是在 IP 包被执行 DNAT 操作之前`--set-xmark 0x4000/0x4000` 打上去的。不过，为什么一定要对流出的包做 SNAT操作呢？

```
            client
              \ ^
               \ \
                v \
node 1   <---   node 2
  | ^    SNAT
  | |    --->
  v |
endpoint
```

当一个外部的 client 通过 node 2 的地址访问一个 Service 的时候，node 2 上的负载均衡规则，就可能把这个 IP 包转发给一个在 node 1 上的 Pod。这里没有任何问题。而当 node 1 上的这个 Pod 处理完请求之后，它就会按照这个 IP 包的源地址发出回复。

如果没有做 SNAT 操作的话，这时候，被转发来的 IP 包的源地址就是 client 的 IP 地址。所以此时，Pod 就会直接将回复发给client。对于 client 来说，它的请求明明发给了 node 2，收到的回复却来自 node 1，这个 client 很可能会报错。

所以，在上面的示意图中，当 IP 包离开 node 2 之后，它的源 IP 地址就会被 SNAT 改成 node 2 的 CNI 网桥地址或者 node 2 自己的地址。这样，Pod 在处理完成之后就会先回复给 node 2（而不是 client），然后再由node 2 发送给 client。



当然，这也就意味着这个 Pod 只知道该 IP 包来自于 node 2，而不是外部的 client。对于 **Pod需要明确知道所有请求来源的场景**来说，这是不可以的。这时，可以将 Service 的 `spec.externalTrafficPolicy` 字段设置为 `local`，这就保证了所有 Pod 通过 Service 收到请求之后，一定可以看到真正的、外部 client 的源地址。

这个机制的实现原理也非常简单：这时候，一台宿主机上的 iptables 规则，会设置为**只将 IP包转发给运行在这台宿主机上的 Pod**。所以这时候，Pod 就可以直接使用源地址将回复包发出，不需要事先进行 SNAT 了。这个流程，如下所示：

```
        client
      ^ /     \
     / /       \
    / v         X
  node 1       node 2
   ^ |
   | |
   | v
 endpoint
```

这也就意味着如果在一台宿主机上，没有任何一个被代理的 Pod 存在，比如上面的 node 2，那么使用 node 2 的 IP 地址访问这个 Service，就是无效的。此时，请求会直接被 DROP 掉。



### LoadBalancer

从外部访问 Service 的第二种方式，适用于公有云上的 Kubernetes 服务，即指定一个 LoadBalancer 类型的 Service。

```
kind: Service
apiVersion: v1
metadata:
  name: example-service
spec:
  ports:
  - port: 8765
    targetPort: 9376
  selector:
    app: example
  type: LoadBalancer
```

在公有云提供的 Kubernetes 服务里，都使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。所以，在上述 LoadBalancer 类型的 Service 被提交后，Kubernetes就会**调用 CloudProvider 在公有云上创建一个负载均衡服务，并且把被代理的 Pod 的 IP地址配置给负载均衡服务做后端。**



###  ExternalName

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

这个 YAML 文件里不需要指定 `selector`，因为它定义的是service 与 domain 间的关联。当通过 Service 的 DNS 名字访问它的时候，比如访问：`my-service.prod.svc.cluster.local`，Kubernetes 返回的就是 `my.database.example.com`。

所以说，ExternalName 类型的 Service，其实是在 kubedns 里添加了一条 CNAME 记录。这时，访问 `my-service.prod.svc.cluster.local` 就和访问 `my.database.example.com` 这个域名是一个效果了。



### 为service 分配 IP 地址

此外，Kubernetes 的 Service 还允许为 Service 分配公有 IP 地址

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```

这样可以通过访问 80.11.12.10:80 访问到被代理的 Pod 。不过，在这里 Kubernetes 要求 **`externalIPs `必须是至少能够路由到一个 Kubernetes 的节点**。



## Ingress

对于LoadBalancer 类型的 Service，它会在 Cloud Provider（比如：Google Cloud 或者OpenStack）里创建一个与该Service 对应的负载均衡服务。由于每个 Service 都要有一个负载均衡服务，所以这个做法实际上既浪费成本又高。作为用户，我其实更希望看到 Kubernetes 为我内置一个全局的负载均衡器。然后，通过我访问的 URL，把请求转发给不同的后端 Service。

而 Ingress 服务是为了 **全局的、代理不同后端 Service 而设置的负载均衡服务**，它的功能其实很容易理解：所谓 Ingress，就是 Service 的“Service”。

它的使用流程也很简单：

1. 创建 Ingress Controller
2. 创建 Service 暴露 Ingress Controller 提供的服务



### 示例

假设现在一个站点：https://cafe.example.com，其中，https://cafe.example.com/coffee，对应的是“咖啡点餐系统”。而，https://cafe.example.com/tea，对应的则是“茶水点餐系统”。这两个系统，分别由名叫 coffee 和 tea 这样两个 Deployment 来提供服务。

> 这个示例的所有代码在这[下载](https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example)。

使用 Kubernetes 的 Ingress 来创建一个统一的负载均衡器，从而实现当用户访问不同的URL 时，能够访问到不同的 Deployment 。

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

在这个名叫 `cafe-ingress.yaml` 文件中，`rules `字段最重要，在Kubernetes 里，这个字段叫作：**IngressRule**。

IngressRule 的 Key，就叫做：`host`。它必须是一个标准的域名格式（Fully Qualified Domain Name）的字符串，而不能是 IP 地址。

而 `host `字段定义的值，就是这个 Ingress 的入口。这也就意味着，当用户访问 cafe.example.com 的时候，实际上访问到的是这个 Ingress 对象。这样，Kubernetes 就能使用 IngressRule 来对请求进行下一步转发。 IngressRule 规则的定义依赖于 `path `字段。可以简单地理解为，这里的每一个 `path `都对应一个后端 Service。在这个例子里，定义了两个 path，它们分别对应 coffee 和 tea 这两个 Deployment 的 Service（即：coffee-svc 和 tea-svc）。

还可以定义多个 host，比如restaurant.example.com、movie.example.com等等，来为更多的域名提供负载均衡服务。

> 所谓 Ingress 对象，其实就是 Kubernetes 项目对“**反向代理**”的一种抽象。
>
> 一个 Ingress 对象的主要内容，实际上就是一个“反向代理”服务（比如：Nginx）的配置文件的描述。而这个代理服务对应的转发规则，就是 IngressRule。



在实际的使用中，只需要从社区里选择一个具体的 Ingress Controller，把它部署在 Kubernetes 集群里即可。然后，这个 Ingress Controller 会根据你定义的 Ingress 对象，提供对应的代理能力。

> 业界常用的各种反向代理项目，比如 Nginx、HAProxy、Envoy、Traefik 等，都已经为Kubernetes 专门维护了对应的 Ingress Controller。



以最常用的 [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx) 为例，进行Ingress 的使用：

```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.25.0/deploy/static/mandatory.yaml
```

这个 YAML 文件定义了一个使用 nginx-ingress-controller 镜像的Pod，这个 Pod 的启动命令需要使用该 Pod 所在的Namespace 作为参数。而这个信息，是通过 Downward API 拿到的，即：Pod 的 env 字段里的定义（`env.valueFrom.fieldRef.fieldPath`）。而这个 Pod 本身，就是一个**监听 Ingress 对象以及它所代理的后端 Service 变化的控制器**。

当一个新的 Ingress 对象由用户创建后，nginx-ingress-controller 就会根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件（`/etc/nginx/nginx.conf`），并使用这个配置文件启动一个 Nginx 服务。

而一旦 Ingress 对象被更新，nginx-ingress-controller 就会更新这个配置文件。但如果只是被代理的 Service 对象被更新，nginx-ingress-controller 所管理的 Nginx服务是不需要重新加载（reload）的，这是因为 nginx-ingress-controller 通过Nginx Lua方案实现了 Nginx Upstream 的动态配置。

此外，nginx-ingress-controller 还允许通过 Kubernetes 的 ConfigMap 对象来对上述 Nginx 配置文件进行定制。这个 ConfigMap 的名字，需要以参数的方式传递给 nginx ingress-controller。在这个 ConfigMap 里添加的字段，将会被合并到最后生成的 Nginx配置文件当中。

可以看到，一个 Nginx Ingress Controller 为你提供的服务，其实是一个可以根据 Ingress对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器。



为了让用户能够用到这个 Nginx，还需要创建一个 Service 来把 Nginx Ingress Controller 管理的 Nginx 服务暴露出去，如下所示：

```
$ kubectl apply -f https://github.com/kubernetes/ingress-nginx/blob/nginx-0.25.0/deploy/static/provider/baremetal/service-nodeport.yaml

$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.99.111.175   <none>        80:31049/TCP,443:30943/TCP   3m4s

# 将访问入口设置为环境变量
$ IC_IP=192.168.50.10 # 任意一台宿主机的地址
$ IC_HTTPS_PORT=30943 # NodePort 端口
```

在本地实验环境中，这个Service 是NodePort 类型的，它会将所有携带 ingress-nginx 标签的 Pod 的 80 和 433 端口暴露出去。

> 如果是公有云上的环境，需要创建的就是 LoadBalancer 类型的 Service了。



部署 cafe 服务：

```
$ kubectl apply -f cafe-secret.yaml # 要创建 Ingress 所需的 SSL 证书（tls.crt）和密钥（tls.key）
$ kubectl apply -f cafe.yaml
$ kubectl apply -f cafe-ingress.yaml

$ kubectl get ingress
NAME           CLASS    HOSTS              ADDRESS   PORTS     AGE
cafe-ingress   <none>   cafe.example.com             80, 443   31s

$  kubectl describe ingress cafe-ingress
Name:             cafe-ingress
Namespace:        default
Address:
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  cafe-secret terminates cafe.example.com
Rules:
  Host              Path  Backends
  ----              ----  --------
  cafe.example.com
                    /tea      tea-svc:80 (10.244.0.17:80,10.244.1.14:80,10.244.1.15:80)
                    /coffee   coffee-svc:80 (10.244.0.16:80,10.244.1.13:80)
Annotations:        Events:
  Type              Reason  Age   From                      Message
  ----              ------  ----  ----                      -------
  Normal            CREATE  53s   nginx-ingress-controller  Ingress default/cafe-ingress
```



访问服务：

```
$  curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecure
Server address: 10.244.0.16:80
Server name: coffee-755d68dd75-wkg9l
Date: 25/Jul/2020:03:45:43 +0000
URI: /coffee
Request ID: 94737c07b20e9f4ed8c9e6eb9bf5e9ca

$ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/tea --insecure
Server address: 10.244.0.17:80
Server name: tea-6c74d89d87-prwcb
Date: 25/Jul/2020:03:46:31 +0000
URI: /tea
Request ID: e32611f3dfe1c183230aaaa762428630
```

通过响应的Server address 和 Server name 字段，可以发现，Nginx Ingress Controller 创建的 Nginx 负载均衡器，已经成功地将请求转发给了对应的后端 Service。



**如果请求没有匹配到任何一条 IngressRule，那么会发生什么呢？**

首先，既然 Nginx Ingress Controller 是用 Nginx 实现的，那么它当然会返回一个 Nginx的 404 页面。

不过，Ingress Controller 也允许通过 Pod 启动命令里的`–default-backend-service` 参数，设置一条默认规则，比如：`–default-backend-service=nginx-default-backend`。这样，任何匹配失败的请求，就都会被转发到这个名叫 nginx-default-backend 的 Service。

所以，可以通过部署一个专门的 Pod，来为用户返回自定义的 404 页面了。