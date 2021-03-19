# Pod

## 基础操作

### 创建 Pod

编写一个YAML 文件，记录容器的定义、参数、配置，然后就可以通过`kubectl apply -f nginx-deployment.yaml`运行这个起来。相比于`docker run`的方式，通过YAML 文件进行部署是 Kubernetes **声明式 API** 所推荐的使用方法。作为用户，你不必关心当前的操作是创建，还是更新，你执行的命令始终是 `kubectl apply`，而 Kubernetes 则会根据YAML 文件的内容变化，自动进行具体的处理。这个流程的好处是，它有助于帮助开发和运维人员，**围绕着可以版本化管理的 YAML 文件，而不是“行踪不定”的命令行进行协作，从而大大降低开发人员和运维人员之间的沟通成本**。

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
	  app: nginx
  replicas: 2
  template:
	metadata:
	  labels:
		app: nginx
	spec:
	  containers:
	  - name: nginx
		image: nginx:1.17.8
		ports:
		- containerPort: 80
```

这样的一个 YAML 文件，对应到 Kubernetes 中，就是一个 API Object（API 对象）。为这个对象的各个字段填好值并提交给 Kubernetes 之后，Kubernetes 就会负责创建出这些对象所定义的容器或者其他类型的 API 资源。

这个 YAML 文件中的 `kind `字段，指定了这个 API 对象的类型（Type），是一个`Deployment`，**Deployment是一个定义多副本应用（即多个副本 Pod）的对象**，它可以保持Pod 的副本数量，当Pod 出错时，会自动创建一个新的Pod，例如这个YAML 文件中副本数量(`spec.replicas`) 是：2。。它还在 Pod 定义发生变化时，对每个副本进行滚动更新（Rolling Update）。



通过`kubectl create`就可以将这个YAML 文件“运行”起来：

```
$ kubectl create -f nginx-deployment.yaml
deployment.apps/nginx-deployment created

# -l 匹配label
$ kubectl get pods -l app=nginx
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-57f4c486cc-cqm9v   1/1     Running   0          5m48s
nginx-deployment-57f4c486cc-wlvg5   1/1     Running   0          5m48s
```

有两个 Pod 处于 Running 状态，也就意味着这个 Deployment 所管理的 Pod 都处于预期的状态。可以通过`kubectl describe`查看API 对象的细节：

```
$ kubectl describe po nginx-deployment-57f4c486cc-cqm9v
Name:         nginx-deployment-57f4c486cc-cqm9v
Namespace:    default
Priority:     0
Node:         client3/10.0.2.15
Start Time:   Sat, 04 Jul 2020 00:43:03 +0000
Labels:       app=nginx
              pod-template-hash=57f4c486cc
Annotations:  <none>
Status:       Running
IP:           10.244.2.36
IPs:
  IP:           10.244.2.36
Controlled By:  ReplicaSet/nginx-deployment-57f4c486cc
Containers:
  nginx:
    ...
Conditions:
    ...
Volumes:
  ...
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  60m   default-scheduler  Successfully assigned default/nginx-deployment-57f4c486cc-cqm9v to client3
  Normal  Pulled     60m   kubelet, client3   Container image "nginx:1.17.8" already present on machine
  Normal  Created    60m   kubelet, client3   Created container nginx
  Normal  Started    60m   kubelet, client3   Started container nginx
```

最下面的`Events`部分，会记录对 API 对象的所有重要操作，是将来进行 Debug 的重要依据。如果有异常发生，一定要第一时间查看这些 Events，往往可以看到非常详细的错误信息。

比如，对于这个 Pod，我们可以看到它被创建之后，被调度器调度（Successfully assigned）到了 client3，发现指定的镜像已经有了，然后启动了 Pod 里定义的容器（Started container）。



在实际使用 Kubernetes 的过程中，相比于编写一个单独的 Pod 的 YAML 文件，使用一个 `replicas=1` 的 Deployment 是更推荐的做法。

因为使用Deployment 后，当pod 所在的节点出故障的时候，pod 可以调度到健康的节点上，而单独的pod只能在节点健康的情况下由kubelet 保证pod 的健康状况。

同时，使用Deployment 有助于后续的扩容缩容，升级回滚，节点间调度等。



### 获取 Pod 信息

- `kubectl get po -n  ${namespace} -o wide`：打印某个`namespace`下的所有Pods信息
- `kubectl get po --all-namespaces`：打印所有namespace 下的Pods的信息
- `kubectl describe po ${PODNAME} -n  ${namespace}`：打印某个Pod的详细信息



### 升级 Pod

如果要对这个 Nginx 服务进行升级，把它的镜像版本从 1.17.8 升级为 1.17.10，只需要修改YAML 文件：

```
...
	spec:
	  containers:
	  - name: nginx
		image: nginx:1.17.10 # change from 1.17.8 to 1.17.10
		ports:
		- containerPort: 80
```

然后使用`kubectl replace`指令完成这个更新：

```
$ kubectl replace -f nginx-deployment.yaml
```

可以看到新旧两个 Pod，被交替创建、删除，最后剩下的就是新版本的Pod，即**滚动更新**。

> 不管是`kubectl create`或者`kubectl replace`，都可以用`kubectl apply`替代。`kubectl apply`是 Kubernetes“声明式 API”所推荐的使用方法。作为用户，你不必关心当前的操作是创建，还是更新，你执行的命令始终是 `kubectl apply`，而 Kubernetes 则会根据YAML 文件的内容变化，自动进行具体的处理。



### 进入 Pod

跟Docker 提供的`exec`指令一样，通过`kubectl exec`也可以进入到Pod 中（即容器的 Namespace 中）：

```
$ kubectl exec -it nginx-deployment-5c96cff55b-dcrlr -n default -- /bin/bash
error: unable to upgrade connection: pod does not exist
```

这个问题的解决可以参考[这个issue](https://github.com/kubernetes/kubernetes/issues/63702)，即网络原因，需要在worker 节点上修改kubelet 的配置，即往`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`文件中加入下面一行，指定worker 节点IP，然后重启kubelet。

```
Environment="KUBELET_EXTRA_ARGS=--node-ip=<worker IP address>"
```

然后就可以进入Pod 里了：

```
$ kubectl exec -it nginx-deployment-5c96cff55b-dcrlr -n default -- /bin/bash
root@nginx-deployment-5c96cff55b-dcrlr:/#
```



### 删除 Pod

要从Kubernetes 集群中删除这个 Nginx Deployment 的话，只需要执行：

```
$ kubectl delete -f nginx-deployment.yaml
```

如果有Pod 在删除后一直没退出，可以强制删除：

```
$ kubectl delete pod ${PODNAME} --grace-period=0 --force -n ${namespace}
```





## 定义 Pod 

`spec.template`部分是Pod 模版，描述了想要创建的 Pod 的细节。在上面的YAML 文件中，这个 Pod 里只有一个容器，这个容器的镜像`spec.containers.image`是`nginx:1.17.8`，这个容器监听端口`containerPort`是 80。

**Pod 就是 Kubernetes 世界里的“应用”；而一个应用，可以由多个容器组成。**

> 这种使用一种 API 对象（Deployment）管理另一种 API 对象（Pod）的方法，在 Kubernetes 中，叫作“控制器”模式（controller pattern）。在这个例子中，Deployment扮演的正是 Pod 的控制器的角色。

每一个 API 对象都有一个叫作 `Metadata `的字段，这个字段就是 API 对象的“标识”，即元数据，它也是我们从 Kubernetes 里找到这个对象的主要依据。这其中最主要使用到的字段是 Labels。

Labels 就是一组 key-value 格式的标签。像 Deployment 这样的控制器对象，就可以通过这个 `Labels `字段从 Kubernetes 中过滤出它所关心的被控制对象。

在这个例子中，Deployment 会把所有正在运行的、携带`app: nginx`标签的Pod 识别为被管理的对象，并确保这些 Pod 的总数严格等于两个。这个**过滤规则的定义，是在 Deployment 的`spec.selector.matchLabels`字段**，一般称之为：Label Selector。

在 Metadata 中，还有一个与 Labels 格式、层级完全相同的字段叫 `Annotations`，它专门用来携带 key-value 格式的内部信息。所谓内部信息，指的是对这些信息感兴趣的，是Kubernetes 组件本身，而不是用户。所以大多数 Annotations，都是在 Kubernetes 运行过程中，被自动加在这个 API 对象上。

一个 Kubernetes 的 API 对象的定义，大多可以分为 Metadata 和 Spec 两个部分。前者存放的是这个对象的元数据，对所有 API 对象来说，这一部分的字段和格式基本上是一样的；而后者存放的，则是属于这个对象独有的定义，用来描述它所要表达的功能。



### 声明 Volume

在 Kubernetes 中，Volume 是属于 Pod 对象的一部分。所以就需要修改 YAML 文件里的 `template.spec` 字段：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.10
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: nginx-vol
      volumes:
      - name: nginx-vol
        emptyDir: {}
```

在 Deployment 的 Pod 模板部分添加了一个 volumes 字段，定义了这个 Pod 声明的所有 Volume。它的名字叫作 `nginx-vol`，类型是 emptyDir。

emptyDir 就相当于Docker 的隐式 Volume 参数，即不显式声明宿主机目录的 Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。

> Kubernetes 的 emptyDir 类型，只是把 Kubernetes 创建的临时目录作为 Volume 的宿主机目录，交给了 Docker。这么做的原因，是 Kubernetes 不想依赖 Docker 自己创建的那个 _data 目录。

Pod 中的容器，使用的是 `volumeMounts`字段来声明自己要挂载哪个 Volume，并通过`mountPath `字段来定义挂载到容器内的位置。

Kubernetes 也提供了显式的 Volume 定义，它叫做 `hostPath`：

```
...
      volumes:
      - name: nginx-vol
        hostpath: 
          path: /var/date
```

修改完YAML 文件后，通过`apply`指令更新这个Deployment：

```
$ kubectl apply -f nginx-deployment.yaml
```

滚动更新过后，通过`describe`指令查看最新的Pod：

```
$ kubectl describe po nginx-deployment-5c96cff55b-dcrlr
...
Volumes:
  nginx-vol:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  <unset>
...
```



在Pod 运行的宿主机上执行下面的语句可以看到emptyDir Volume对应在宿主机上的文件：

```
$ ls -l /var/lib/kubelet/pods/`kubectl get pod -n ${namespace} ${PODNAME} -o 'jsonpath={.metadata.uid}'`/volumes/kubernetes.io~empty-dir
```



## 为什么需要 Pod

### 进程组

对于Linux 容器来说，Namespace 做隔离，Cgroups 做限制，rootfs 做文件系统，为什么还需要Pod 呢？

首先，容器的本质是进程，容器镜像就相当于进程的静态形式——“.exe”安装包，而Kubernetes 就是操作系统了。

在 Linux 系统中，执行`pstree -g`可以看到当前系统中正在运行的进程的树状结构。在一个真正的操作系统里，进程并不是“孤苦伶仃”地独自运行的，而是以进程组的方式，“有原则的”组织在一起。在进程的树状图中，每一个进程后面括号里的数字，就是它的进程组 ID（Process Group ID, PGID）。

```
$ pstree -g
systemd(1)─┬─AliYunDun(1224)─┬─{AliYunDun}(1224)
			......
           ├─rsyslogd(696)─┬─{in:imklog}(696)
                ├─{in:imuxsock}(696)
                └─{rs:main Q:Reg}(696)
```

以 rsyslogd 的程序为例，它负责的是 Linux 操作系统里的日志处理。可以看到，rsyslogd 的主程序 main，和它要用到的内核日志模块 imklog 等，同属于 696 进程组。这些进程相互协作，共同完成 rsyslogd 程序的职责。

对于操作系统来说，通过进程组的方式 更方便管理。例如，Linux 操作系统只需要将信号，比如 SIGKILL 信号，发送给一个进程组，那么该进程组中的所有进程就都会收到这个信号而终止运行。

而 Kubernetes 所做的，就是**将“进程组”的概念映射到了容器技术中**，并使其成为了这个云计算“操作系统”里的“一等公民”。

如果事先没有“组”的概念，运维起来就会非常难以处理。例如 rsyslogd 由三个进程组成：一个 imklog 模块，一个imuxsock 模块，一个 rsyslogd 自己的 main 函数主进程。这三个进程一定要运行在同一台机器上，否则，它们之间基于 Socket 的通信和文件交换，都会出现问题。

如果以容器的方式部署 rsyslogd 应用，由于受限于容器的“**单进程模型**”，这三个模块必须被分别制作成三个不同的容器。而在这三个容器运行的时候，它们设置的内存配额都是 1 GB。

> **容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。**这是因为容器里 `PID=1` 的进程就是应用本身，其他的进程都是这个`PID=1 `进程的子进程。可是，**用户编写的应用，并不能够像正常操作系统里的 init 进程或者systemd 那样拥有进程管理的功能。**
>
> 比如，容器的应用是一个 Java Web 程序（`PID=1`），通过执行 `docker exec `在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？

假设 Kubernetes 集群上有两个节点：worker1 上有 3 GB 可用内存，worker2 有 2.5 GB 可用内存。

这时，假设用 Docker Swarm 来运行这个 rsyslogd 程序。为了能够让这三个容器都运行在同一台机器上，我就必须在另外两个容器上设置一个 `affinity=main`（与 main 容器有亲和性）的约束，即：它们俩必须和 main 容器运行在同一台机器上。

然后顺序执行：`docker run main; docker run imklog; docker run imuxsock`，创建这三个容器。这样，这三个容器都会进入 Swarm 的待调度队列。然后，main 容器和 imklog 容器都先后出队并被调度到了 worker2 上（这个情况是完全有可能的）。

可是，当 imuxsock 容器出队开始被调度时，worker2 上的可用资源只有 0.5GB 了，并不足以运行 imuxsock 容器；可是，根据 `affinity=main` 的约束，imuxsock 容器又只能运行在 worker2 上。这就是一个典型的**成组调度**（gang scheduling）没有被妥善处理的例子。

> 关于 成组调度 这个问题的讨论，Mesos 中就有一个资源囤积（resource hoarding）的机制，会在所有设置了 Affinity 约束的任务都达到时，才开始对它们统一进行调度。而在 Google Omega 论文中，则提出了使用乐观
> 调度处理冲突的方法，即：先不管这些冲突，而是通过精心设计的回滚机制在出现了冲突之后解决问题。
>
> 不过，这些方法都谈不上完美。资源囤积带来了不可避免的调度效率损失和死锁的可能性；而乐观调度的复杂程度，则不是常规技术团队所能驾驭的。

但是，在 Kubernetes 中，这样的问题就迎刃而解了：Pod 是 Kubernetes 里的原子调度单位，而一个Pod 可以由多个容器组成。这就意味着，**Kubernetes 的调度器，是统一按照 Pod 而非容器的资源需求进行计算的**。

像 imklog、imuxsock 和 main 函数主进程这样的三个容器，正是一个典型的由三个容器组成的 Pod。Kubernetes 项目在调度时，自然就会去选择可用内存等于 3 GB 的 worker1 节点进行绑定，而根本不会考虑 worker2。

这种容器间的紧密协作，可以称为“超亲密关系”。具有“超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。

这也就意味着，并不是所有有“关系”的容器都属于同一个 Pod。比如，PHP 应用容器和 MySQL虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个 Pod。



### 容器设计模式

如果只是处理“超亲密关系”这样的调度问题，Kubernetes 可以在调度器层面给它解决掉，而不需要创造Pod 。Pod 在 Kubernetes 里有更重要的意义，那就是：容器设计模式。

#### Pod 实现原理

首先，关于 Pod 最重要的一个事实是：**它只是一个逻辑概念**。Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和Cgroups，而**并不存在一个所谓的 Pod 的边界或者隔离环境**。

**Pod，其实是一组共享了某些资源的容器，Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个Volume**。

一个有 A、B 两个容器的 Pod，不就等同于一个容器（容器 A）共享另外一个容器（容器 B）的网络和 Volume ，这通过 `docker run --net --volumes-from` 命令就能实现嘛，比如：

```
$ docker run --net=B --volumes-from=B --name=A image-A ..
```

但是，如果真这样做的话，容器 B 就必须比容器 A 先启动，这样一个 Pod 里的多个容器就不是对等关系，而是拓扑关系了。

所以，**在 Kubernetes 里，Pod 的实现需要使用一个中间容器**，这个容器叫作 Infra 容器。在每个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

![Pod 内容器的组织关系](Pod.assets/1593853597343.png)

上图中这个 Pod 里有两个用户容器 A 和 B，还有一个 Infra 容器。很容易理解，在Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像：`k8s.gcr.io/pause`。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有几百 KB。

在 Infra 容器“Hold 住”Network Namespace 后，用户容器就可以加入到 Infra 容器的Network Namespace 当中了。所以，如果查看这些容器在宿主机上的 Namespace 文件，它们指向的值一定是完全一样的。

> Linux Namespaces: mount、pid、network、ipc、uid、UTS、cgroup、time

这就意味着，对于 Pod 里的容器 A 和容器 B 来说：

- 它们可以直接使用 localhost 进行通信
- 它们看到的网络设备跟 Infra 容器看到的完全一样
- 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址
- 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享
- Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关

对于同一个 Pod 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 Infra 容器完成的。这一点很重要，因为将来如果要为 Kubernetes 开发一个网络插件时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。

这意味着，如果网络插件需要在容器里安装某些包或者配置才能完成的话，是不可取的：Infra 容器镜像的 rootfs 里几乎什么都没有，没有你随意发挥的空间。当然，这同时也意味着网络插件完全不必关心用户容器的启动与否，而只需要关注如何配置 Pod，也就是 Infra 容器的 Network Namespace 即可。

有了这个设计之后，共享 Volume 就简单多了：Kubernetes 只要把所有 Volume 的定义都设计在 Pod 层级即可。一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录。

```
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
  hostPath:
    path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```



#### 例子

明白了 Pod 的实现原理后，再来看“容器设计模式”，就容易多了。Pod 这种“超亲密关系”容器的设计思想，实际上就是希望，**当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。**

为了能够掌握这种思考方式，尽量尝试使用它来描述一些用单个容器难以解决的问题。

##### WAR 包与 Web 服务器

假设有一个 Java Web 应用的 WAR 包，它需要被放在 Tomcat 的 `webapps `目录下运行起来。

如果只能用 Docker 来做这件事情，把 WAR 包直接放在 Tomcat 镜像的 `webapps `目录下，做成一个新的镜像运行起来。但是如果要更新 WAR 包的内容，或者升级 Tomcat 镜像，就要重新制作一个新的发布镜像，非常麻烦。

那如果不考虑 WAR 包，永远只发布一个 Tomcat 容器。不过，这个容器的`webapps `目录，就必须声明一个 `hostPath `类型的 Volume，从而把宿主机上的 WAR 包挂载进Tomcat 容器当中运行起来。不过，这样就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有 WAR 包的目录呢？这样子只能独立维护一套分布式存储系统了。



而通过Pod 来实现这个功能就很简单，把WAR 包和 Tomcat 分别做成两个镜像，然后用一个`initContainer`把WAR 包提前放到Tomcat 的`webapps`目录下，最后再启动Tomcat 容器：

```
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: xxx/cpWar:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: xxx/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/path-to-tomcat/bin/start.sh"]
    volumeMounts:
    - mountPath: /path-to-tomcat/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001
  volumes:
  - name: app-volume
    emptyDir: {}
```

在 Pod 中，所有 `Init Container` 定义的容器，都会比 `spec.containers` 定义的用户容器先启动。并且，**`Init Container `容器会按顺序逐一启动**，而直到它们都启动并且退出了，用户容器才会启动。

因此，等 Tomcat 容器启动时，它的 `webapps `目录下就一定会存在 `sample.war` 文件：这个文件正是 WAR 包容器启动时拷贝到这个 Volume 里面的，而这个 Volume 是被这两个容器共享的。

这种组合方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。实际上，这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式：**sidecar**。顾名思义，**sidecar 指的就是可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。**

比如，在这个应用 Pod 中，Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用 Init Container 的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色。



##### 日志收集

假如现在有一个应用，需要不断地把日志文件输出到容器的 `/var/log` 目录中。这时就可以把一个 Pod 里的 Volume 挂载到应用容器的 `/var/log` 目录上。然后，在这个 Pod 里同时运行一个 sidecar 容器，它也声明挂载同一个 Volume 到自己的 `/var/log` 目录上。

接下来 sidecar 容器就只需要做一件事：不断地从自己的 `/var/log `目录里读取日志文件，转发到 MongoDB 或者 Elasticsearch 中存储起来。这样，一个最基本的日志收集工作就完成了。

跟上一个例子一样，这个例子中的 sidecar 的主要工作也是使用共享的 Volume 来完成对文件的操作。

但不要忘记，Pod 的另一个重要特性是，它的所有容器都共享同一个 Network Namespace。这就使得很多与 Pod 网络相关的配置和管理，也都可以交给 sidecar 完成，而完全无须干涉用户容器。这里最典型的例子莫过于 Istio 这个微服务治理项目了。





## Pod 的属性

Pod，而不是容器，是 Kubernetes 中的最小编排单位。将这个设计落实到 API 对象上，容器（Container）就成了 Pod 属性里的一个普通的字段。那么，到底哪些属性属于 Pod 对象，而又有哪些属性属于 Container 呢？

Pod 相当于传统部署环境里“虚拟机”的角色，这样的设计，是为了使用户从传统环境（虚拟机环境）向Kubernetes（容器环境）的迁移，更加平滑。

如果把 Pod 看成传统环境里的“机器”、把容器看作是运行在这个“机器”里的“用户程序”，那么很多关于 Pod 对象的设计就非常容易理解。

**凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。**这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。比如，配置这个“机器”的网卡（即：Pod 的网络定义），配置这个“机器”的磁盘（即：Pod 的存储定义），配置这个“机器”的防火墙（即：Pod 的安全定义），以及 这台“机器”运行在哪个服务器之上（即：Pod 的调度）。



### NodeSelector

这是一个**供用户将Pod 与 Node 进行绑定的字段**：

```
apiVersion: v1
kind: Pod
...
spec:
  nodeSelector:
    disktype: ssd
```

这样的配置，意味着这个 Pod 永远只能运行在携带了`disktype: ssd`标签（Label）的节点上；否则，它将调度失败。



### NodeName

一旦Pod 的这个字段被赋值，Kubernetes 就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。



### HostAliases

定义了 Pod 的 hosts 文件（比如 `/etc/hosts`）里的内容：

```
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

这个 Pod 启动后，`/etc/hosts` 文件的内容将如下所示：

```
$ cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

最下面两行记录，就是通过 `hostAliases` 字段为 Pod 设置的。

**在Kubernetes 中，如果要设置 hosts 文件里的内容，一定要通过这种方法。**否则，如果直接修改 hosts 文件，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。



### Linux Namespace

除了上述跟“机器”相关的配置外，凡是跟容器的 Linux Namespace 相关的属性，也一定是 Pod 级别的。因为Pod 的设计，就是要让它里面的容器尽可能多地共享 Linux Namespace，仅保留必要的隔离和限制能力。这样，Pod 模拟出的效果，就跟虚拟机里程序间的关系非常类似了。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

上面的YAML 文件定义了`shareProcessNamespace=true`，这就意味着这个 Pod 里的容器要共享 PID Namespace。



类似地，**凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义**。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

在这个 Pod 中，定义了共享宿主机的 Network、IPC 和 PID Namespace。这就意味着，这个Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程。



### Containers

Containers 是Pod 最重要的一个字段了，它跟Init Containers 一样，都属于 Pod 对容器的定义，内容也完全相同，只是 Init Containers 的生命周期，会先于所有的 Containers，并且严格按照定义的顺序执行。

Kubernetes 中对 Container 的定义，和 Docker 相比并没有什么太大区别。不过，有几个新的属性。

#### ImagePullPolicy

这个属性定义了镜像拉取的策略，它之所以是一个 Container 级别的属性，是因为容器镜像本来就是 Container 定义中的一部分。

`ImagePullPolicy `的值默认是 `Always`，即每次创建 Pod 都重新拉取一次镜像。另外，当容器的镜像是类似于 `nginx `或者 `nginx:latest` 这样的名字时，`ImagePullPolicy `也会被认为 `Always`。

而如果它的值被定义为 `Never `或者 `IfNotPresent`，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。



#### Lifecycle

定义了 Container Lifecycle Hooks，即在容器状态发生变化时触发的一系列“钩子”。

```
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

这个YAML 文件只定义了一个 nginx 镜像的容器。不过，在这个 YAML 文件的容器部分，设置了一个 `postStart `和 `preStop `参数。

`postStart `指的是，**在容器启动后，立刻执行一个指定的操作**。需要明确的是，`postStart `定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 `postStart `启动时，ENTRYPOINT 有可能还没有结束。

如果 `postStart `执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。

类似地，`preStop `发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。**`preStop `操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 `postStart `不一样。**

所以，在这个例子中，在容器成功启动之后，在` /usr/share/message` 里写入了一句“欢迎信息”（即 postStart 定义的操作）。而在这个容器被删除之前，则先调用了 nginx 的退出指令（即 `preStop `定义的操作），从而实现了容器的“优雅退出”。



### 生命周期

Pod 在 Kubernetes 中的生命周期的变化，主要体现在 Pod API 对象的`Status `部分，这是它除了 `Metadata `和 `Spec` 之外的第三个重要字段。其中，`pod.status.phase`，就是 Pod 的当前状态，它有如下几种可能的情况：
1. `Pending`：这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。
2. `Running`：这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
3. `Succeeded`：这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
4. `Failed`：这个状态下，Pod 里至少有一个容器以不正常的状态（非 0 的返回码）退出。这个状态的出现，意味着你得想办法 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. `Unknown`：这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kubeapiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

更进一步地，Pod 对象的 `Status `字段，还可以再细分出一组 `Conditions`。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及 Unschedulable。它们主要用于**描述造成当前Status 的具体原因是什么**。

比如，Pod 当前的 Status 是 Pending，对应的 Condition 是 Unschedulable，这就意味着它的调度出现了问题。

**Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了**。这两者之间（Running 和 Ready）是有区别的。





## Projected Volume

在 Kubernetes 中，有几种特殊的 Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊 Volume 的作用，是**为容器提供预先定义好的数据**。所以，从容器的角度来看，这些 Volume 里的信息仿佛是被 Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume 的含义。

Kubernetes 支持的 Projected Volume 一共有四种：

- [`secret`](https://kubernetes.io/docs/concepts/storage/volumes/#secret)
- [`downwardAPI`](https://kubernetes.io/docs/concepts/storage/volumes/#downwardapi)
- [`configMap`](https://kubernetes.io/docs/concepts/storage/volumes/#configmap)
- `serviceAccountToken`



### secret

secret 的作用是：把 Pod 想要访问的加密数据，存放到 Etcd 中。然后就可以通过在 Pod 的容器里挂载 Volume 的方式，访问到这些 Secret里保存的信息了。

#### 创建

在 username.txt 和 password.txt 文件里存放用户名和密码，然后创建名为 user 和 pass 的 Secret 对象：

```
$ cat ./username.txt
admin
$ cat ./password.txt
123
$ kubectl create secret generic user --from-file=./username.txt
secret/user created
$ kubectl create secret generic pass --from-file=./password.txt
secret/pass created
```

除了通过`kubectl create secret`的方式创建secret，也可以通过YAML 文件的方式`kubectl apply -f mysecret.yaml`创建：

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MTIz
```

这个文件创建了名为`mysecret`的secret 对象，它的`data`字段以键值对的方式存放了两份secret 数据。**Secret 对象要求这些数据必须是经过 Base64 转码的**，以免出现明文密码的安全隐患。这个转码操作也很简单：

```
# -n 不输出换行符
$ echo -n admin|base64
YWRtaW4=
$ echo -n 123|base64
MTIz
```

> 这样创建的 Secret 对象，它里面的内容仅仅是经过了转码，并没有被加密。在真正的生产环境中，还需要开启 Kubernetes 中的 Secret 加密插件，增强数据的安全性。



#### 使用

Secret 最典型的使用场景，就是存放数据库的 Credential 信息了：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
    - name: mysecret
      mountPath: "/mysecret"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
  - name: mysecret
    projected:
      sources:
      - secret:
          name: mysecret
```

这个 Pod 定义了一个简单的容器。它声明挂载的 Volume，并不是常见的 `emptyDir `或者 `hostPath `类型，而是 `projected `类型。而这个 Volume 的数据来源（sources），则是名为 user和 pass 的 Secret 对象，分别对应的是数据库的用户名和密码。



现在可以通过`kubectl get secrets`查看已有的secrets：

```
$ kubectl get secretsant#
NAME                  TYPE                                  DATA   AGE
default-token-l489d   kubernetes.io/service-account-token   3      3d14h
mysecret              Opaque                                2      5m47s
pass                  Opaque                                1      39m
user                  Opaque                                1      39m
```

创建Pod 并进入到Pod 中查看 secret：

```
$ kubectl apply -f test-projected-volume.yml
$ kubectl exec -it  test-projected-volume -- /bin/sh
/ ls /projected-volume
password.txt  username.txt
/ # cat /projected-volume/username.txt
admin
/ # cat /projected-volume/password.txt
123

/ # cat /mysecret/pass
123
/ # cat /mysecret/user
admin/
```

**通过挂载方式进入到容器里的 Secret，一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新。**这是因为 kubelet 组件在定时维护这些Volume。

不过这个更新可能会有一定的延时，所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是个好习惯。



### configMap

ConfigMap 与 Secret 类似 ，它与 Secret 的区别在于，ConfigMap 保存的是不需要加密的、应用所需的配置信息。而 ConfigMap 的用法几乎与 Secret 完全相同：可以使用 `kubectl create configmap` 从文件或者目录创建 ConfigMap，也可以直接编写 ConfigMap 对象的 YAML 文件。

以一个 Java 应用所需的`.properties` 配置文件为例，可以通过下面这样的方式保存在 ConfigMap 里：

```
# .properties 文件的内容
$ cat <<EOF > test.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
EOF

# 从.properties 文件创建 ConfigMap
$ kubectl create configmap test --from-file=./test.properties
configmap/test created

# 查看这个 ConfigMap 里保存的信息 (data)
$ kubectl get cm test -o yaml
apiVersion: v1
data:
  test.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
......
```



### downwardApi

这个Volume的作用是**让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息**，包括Pod 的信息和 容器的信息。可以获取的[字段](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/#capabilities-of-the-downward-api)如下：

- `fieldRef`：
  - `metadata.name` - Pod 的名字
  - `metadata.namespace` - Pod 的namespace
  - `metadata.uid` - Pod 的 UID
  - `metadata.labels['<KEY>']` - 指定`<KEY>` 的Label 值，例如 `metadata.labels['mylabel']`
  - `metadata.annotations['<KEY>']` -  指定`<KEY>` 的annotation值，例如``metadata.annotations['myannotation']`
  - `metadata.labels` - 所有label， 以`label-key="escaped-label-value"`形式一行一个label
  - `metadata.annotations` - 所有annotation，以`annotation-key="escaped-annotation-value"`形式一行一个
  - `status.podIP` - Pod 的 IP
  - `status.hostIP` - 节点的IP
  - `spec.serviceAccountName` - Pod 的 Service Account 的名字
  - `spec.nodeName` - 节点的名字
- `resourceFieldRef`：
  - A Container’s CPU limit
  - A Container’s CPU request
  - A Container’s memory limit
  - A Container’s memory request
  - A Container’s ephemeral-storage limit
  - A Container’s ephemeral-storage request

下面的这个YAML 文件读取Downward Api 挂载的volume，并打印读取到的信息：

```
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          if [[ -e /etc/podinfo/annotations ]]; then
            echo -en '\n\n'; cat /etc/podinfo/annotations; fi;
          if [[ -e /etc/podinfo/cpu_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_limit; fi;
          if [[ -e /etc/podinfo/cpu_request ]]; then
            echo -en '\n'; cat /etc/podinfo/cpu_request; fi;
          if [[ -e /etc/podinfo/mem_limit ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_limit; fi;
          if [[ -e /etc/podinfo/mem_request ]]; then
            echo -en '\n'; cat /etc/podinfo/mem_request; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.cpu
              divisor: 1m
          - path: "cpu_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.cpu
              divisor: 1m
          - path: "mem_limit"
            resourceFieldRef:
              containerName: client-container
              resource: limits.memory
              divisor: 1Mi
          - path: "mem_request"
            resourceFieldRef:
              containerName: client-container
              resource: requests.memory
              divisor: 1Mi
```

创建Pod 并查看其log，到`/etc/podinfo`路径下看看有什么内容：

```
$ kubectl apply -f kubernetes-downwardapi-volume-example.yaml
$ kubectl logs kubernetes-downwardapi-volume-example
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"

build="two"
builder="john-doe"
...
2000
0
1893
0

$  kubectl exec -it kubernetes-downwardapi-volume-example -- /bin/sh
/ # ls /etc/podinfo
annotations  cpu_limit    cpu_request  labels       mem_limit    mem_request
/ # cat /etc/podinfo/labels
cluster="test-cluster1"
rack="rack-22"
```

**Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。**如果想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 sidecar 容器。

其实，Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过环境变量的方式出现在容器里。但是，**通过环境变量获取这些信息的方式，不具备自动更新的能力。**所以，一般情况下，建议使用 Volume 文件的方式获取这些信息。



### serviceAccountToken

能不能在这个 Pod 里安装一个 Kubernetes 的 Client，这样就可以从容器里直接访问并且操作这个 Kubernetes 的 API 了呢？

这当然是可以的。不过，首先要解决 **API Server 的授权问题**。

**Service Account 对象的作用，就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes进行权限分配的对象。**比如，Service Account A，可以只被允许对 Kubernetes API 进行 GET 操作，而 Service Account B，则可以有 Kubernetes API 的所有操作的权限。

像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作ServiceAccountToken。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，才可以合法地访问 API Server。

所以说，Kubernetes 项目的 Projected Volume 其实只有三种，因为**ServiceAccountToken，只是一种特殊的 Secret 而已。**

为了方便使用，Kubernetes 已经提供了一个的默认“服务账户”（default Service Account）。并且，**任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它。**

这是通过 Projected Volume 机制实现的。`kubectl describe`任意一个运行在 Kubernetes 集群里的 Pod，就会发现，这一个 Pod，都已经自动声明一个类型是 Secret、名为 `default-token-xxxx` 的 Volume，然后 自动挂载在每个容器的一个固定目录上。比如：

```
Containers:
...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-l489d (ro)
Volumes:
  default-token-l489d:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-l489d
    Optional:    false
```

这个 Secret 类型的 Volume，正是默认 Service Account 对应的 ServiceAccountToken。

> Kubernetes 其实在每个 Pod 创建的时候，自动在它的 `spec.volumes` 部分添加上了默认ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 `volumeMounts `字段。这个过程对于用户来说是完全透明的。

这样，一旦 Pod 创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。这个容器内的路径在 Kubernetes 里是固定的，即：`/var/run/secrets/kubernetes.io/serviceaccount `，而这个 Secret 类型的 Volume 里面的内容如下所示：

```
/ # ls -al /var/run/secrets/kubernetes.io/serviceaccount
total 4
drwxrwxrwt    3 root     root           140 Jul  7 04:59 .
drwxr-xr-x    3 root     root          4096 Jul  7 04:59 ..
drwxr-xr-x    2 root     root           100 Jul  7 04:59 ..2020_07_07_04_59_36.487127094
lrwxrwxrwx    1 root     root            31 Jul  7 04:59 ..data -> ..2020_07_07_04_59_36.487127094
lrwxrwxrwx    1 root     root            13 Jul  7 04:59 ca.crt -> ..data/ca.crt
lrwxrwxrwx    1 root     root            16 Jul  7 04:59 namespace -> ..data/namespace
lrwxrwxrwx    1 root     root            12 Jul  7 04:59 token -> ..data/token
```

所以，应用程序只要直接加载这些授权文件，就可以访问并操作 Kubernetes API 了。而且，如果使用 Kubernetes 官方的 Client 包（`k8s.io/client-go`）的话，它还可以自动加载这个目录下的文件，而不需要做任何配置或者编码操作。

这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作**InClusterConfig**，也是最推荐的进行 Kubernetes API 编程的授权方式。

> 考虑到自动挂载默认 ServiceAccountToken 的潜在风险，Kubernetes 允许设置默认不为 Pod 里的容器自动挂载这个 Volume。

除了这个默认的 Service Account 外，很多时候还需要创建一些自定义的 Service Account，来对应不同的权限设置。这样，Pod 里的容器就可以通过挂载这些 Service Account 对应的 ServiceAccountToken，来使用这些自定义的授权信息。



## 容器健康检查和恢复机制

### livenessProbe

在 Kubernetes 中，可以为 Pod 里的容器定义一个健康检查“探针”（Probe）。这样，kubelet 就会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器进行是否运行（来自 Docker 返回的信息）作为依据。这种机制，是生产环境中保证应用健康存活的重要手段。

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

这个 Pod 定义了一个容器，这个容器启动之后就在 `/tmp` 目录下创建了一个 `healthy`文件，以此作为自己已经正常运行的标志。而 30 s 过后，它会把这个文件删除掉。

同时，定义了一个 `livenessProbe`（健康检查），它的类型是 `exec`，这意味着，它会在容器启动后，在容器里面执行我们指定的命令，比如：`cat /tmp/healthy`。这时，如果这个文件存在，这条命令的返回值就是 0，Pod 就会认为这个容器不仅已经启动，而且是健康的。这个健康检查，在容器启动 5 s 后开始执行（`initialDelaySeconds`: 5），每 5 s 执行一次（`periodSeconds`: 5）。

除了在容器中执行命令外，`livenessProbe `也可以定义为发起 HTTP 或者 TCP 请求的方式，定义格式如下：

```
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: Awesome
  initialDelaySeconds: 3
  periodSeconds: 3
  
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
```

运行这个Pod，30秒后查看其Events：

```
$ kubectl apply -f liveness-exec.yaml
$ kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
liveness-exec           1/1     Running   0          11s

# 30秒后查看Events
$ kubectl describe po liveness-exec
...
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Warning  Unhealthy  4s (x2 over 9s)  kubelet, client2   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

这个 Pod 在 Events 报告了一个异常，即这个健康检查探查到` /tmp/healthy `已经不存在了，所以它报告容器是不健康的。再次查看这个Pod 的状态：

```
$ kubectl get po
NAME                    READY   STATUS    RESTARTS   AGE
liveness-exec           1/1     Running   1          2m28s
```

Pod 并没有进入 Failed 状态，而是保持了 Running 状态。 但是 `RESTARTS `字段从 0 变成了 1 ，说明这个异常的容器已经被Kubernetes 重启了。在这个过程中，Pod 保持 Running 状态不变。

> Kubernetes 中并没有 Docker 的 Stop 语义。所以虽然是 Restart（重启），但**实际却是重新创建了容器**。

这个功能就是 Kubernetes 里的Pod 恢复机制，也叫 `restartPolicy`。它是 Pod 的 `Spec `部分的一个标准字段`pod.spec.restartPolicy`，默认值是 `Always`，即：任何时候这个容器发生了异常，它一定会被重新创建。

但一定要强调的是，**Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。**事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（`pod.spec.node` 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。

**如果想让 Pod 出现在其他的可用节点上，就必须使用 Deployment 这样的“控制器”来管理 Pod，哪怕只需要一个 Pod 副本。**



### restartPolicy

`restartPolicy`，除了 `Always`，它还有`OnFailure `和 `Never `两种情况：

- `Always`：在任何情况下，只要容器不在运行状态，就自动重启容器
- `OnFailure`: 只在容器 异常时才自动重启容器
- `Never`: 从来不重启容器

在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。

比如，一个 Pod，只计算 1+1=2，计算完成输出结果后退出，变成 Succeeded 状态。这时，再用`restartPolicy=Always`  强制重启这个 Pod 的容器，就没有任何意义了。

如果关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将`restartPolicy `设置为 `Never`。因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被垃圾回收了）。



Kubernetes 的官方文档，把 `restartPolicy `和 Pod 里容器的状态，以及 Pod 状态的对应关系，进行了[总结](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#example-states)，基本的设计原理如下：

1. **只要 Pod 的 `restartPolicy `指定的策略允许重启异常的容器（比如：`Always`），那么这个 Pod就会保持 Running 状态，并进行容器重启。**否则，Pod 就会进入 `Failed `状态 。

2. **对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态。**在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数，比如：

   ```
   $ kubectl get po
   NAME                    READY   STATUS    RESTARTS   AGE
   liveness-exec           0/1     Running   1          5m
   ```

   

假如一个 Pod 里只有一个容器，然后这个容器异常退出了。那么，只有当`restartPolicy=Never` 时，这个 Pod 才会进入 Failed 状态。而其他情况下，由于 Kubernetes 都可以重启这个容器，所以 Pod 的状态保持 Running 不变。

而如果这个 Pod 有多个容器，仅有一个容器异常退出，它就始终保持 Running 状态，哪怕即使`restartPolicy=Never`。只有当所有容器也异常退出之后，这个 Pod 才会进入 Failed 状态。



### readinessProbe

在 Kubernetes 的 Pod 中，还有一个叫 `readinessProbe `的字段。虽然它的用法与 `livenessProbe`类似，但作用却大不一样。`readinessProbe `检查结果的成功与否，决定的是**这个 Pod 是不是能被通过 Service 的方式访问到**，而并不影响 Pod 的生命周期。



## 为Pod 自动填充字段

Pod 有这么多的字段，如果开发人员只需要提交一个基本的、非常简单的 Pod YAML，Kubernetes 就可以自动给对应的 Pod 对象加上其他必要的信息，比如 labels，annotations，volumes 等等。而这些信息，可以是运维人员事先定义好的。这么一来，开发人员编写 Pod YAML 的门槛，就被大大降低了。

在v1.11 版本的kubernetes 中， 出现了 PodPreset 的功能。编写一份简单的YAML 文件：

```
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
  - name: website
    image: nginx
    ports:
    - containerPort: 80
```

定义一个 PodPreset 对象。在这个对象中，凡是想在开发人员编写的 Pod 里追加的字段，都可以预先定义好。比如这个 `preset.yaml`：

```
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

在这个 PodPreset 的定义中，首先是一个 selector。这就意味着后面这些追加的定义，只会作用于 selector 所定义的、带有`role: frontend`标签的 Pod 对象，这就可以防止“误伤”。

然后定义了一组 Pod 的 `Spec `里的标准字段，以及对应的值。比如，env 里定义了`DB_PORT `这个环境变量，`volumeMounts `定义了容器 Volume 的挂载目录，`volumes `定义了一个 `emptyDir `的 Volume。

接下来，先创建这个 PodPreset，然后再创建 Pod：

```
$ kubectl apply -f preset.yaml
$ kubectl apply -f pod.yaml

$ kubectl get podpreset
NAME             CREATED AT
allow-database   2020-07-07T08:54:29Z

$ kubectl get pod website -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
  annotations:
    podpreset.admission.kubernetes.io/podpreset-allow-database: "resource version"
spec:
  containers:
    - name: website
      image: nginx
      volumeMounts:
        - mountPath: /cache
          name: cache-volume
      ports:
        - containerPort: 80
      env:
        - name: DB_PORT
          value: "6379"
  volumes:
    - name: cache-volume
      emptyDir: {}
```

可以看到，这个 Pod 里多了新添加的 labels、env、volumes 和volumeMount 的定义，它们的配置跟 PodPreset 的内容一样。此外，这个 Pod 还被自动加上了一个 annotation 表示这个 Pod 对象被 PodPreset 改动过。

**PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。**比如，现在提交一个 nginx-deployment，那么这个 Deployment 对象本身是永远不会被PodPreset 改变的，被修改的只是这个 Deployment 创建出来的所有 Pod。

如果定义了同时作用于一个 Pod 对象的多个 PodPreset，Kubernetes 会合并（Merge）这两个 PodPreset 要做的修改。而如果它们要做的修改有冲突的话，这些冲突字段就不会被修改。

`preset.yaml`中定义了`name`为`cache-volume`的volume，挂载到了`/cache`，如果`pod.yaml`中也定义了某个volume挂载到`/cache`，那么就和`preset.yaml`冲突，会导致`preset.yaml`的内容全都不会应用到新创建的Pod中，即新创建的Pod 中将没有对应的`annotations`。





