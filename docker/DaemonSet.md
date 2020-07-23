# DaemonSet

DaemonSet 可以让你在 Kubernetes 集群里，运行一个 Daemon Pod，这个 Pod 有如下三个特征：

1. 这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上
2. 每个节点上只有一个这样的 Pod 实例
3. 当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉

虽然 DaemonSet 的机制很简单，但它的意义很重要，它是容器化的守护进程，可以用在：

1. 各种网络插件的 Agent 组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；
2. 各种存储插件的 Agent 组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的 Volume 目录；
3. 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。



更重要的是，跟其他编排对象不一样，**DaemonSet 开始运行的时机，很多时候比整个 Kubernetes 集群出现的时机都要早。**

假如有一个 DaemonSet 是一个网络插件的 Agent 组件，这个时候，整个 Kubernetes 集群里还没有可用的容器网络，所有 Worker 节点的状态都是 `NotReady`（`NetworkReady=false`）。这种情况下，普通的 Pod 肯定不能运行在这个集群上。所以，这也就意味着 DaemonSet 的设计，必须要有某种“过人之处”才行。



## 定义

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

这个 DaemonSet，管理了一个 fluentd-elasticsearch 镜像的 Pod。这个镜像的功能非常实用：通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中。

DaemonSet 跟 Deployment 其实非常相似，只是没有 `replicas `字段；它也使用 `selector `选择管理所有携带了`name=fluentd-elasticsearch` 标签的 Pod。这些 Pod 的模板，也是用 `template `字段定义的。

容器挂载了两个 `hostPath `类型的 Volume，分别对应宿主机的 `/var/log` 目录和 `/var/lib/docker/containers `目录。fluentd 启动之后，它会从这两个目录里搜集日志信息，并转发给 ElasticSearch 保存。这样，我们通过 ElasticSearch 就可以很方便地检索这些日志了。

> Docker 容器里应用的日志，默认会保存在宿主机的 `/var/lib/docker/containers/{{. 容器 ID}}/{{. 容器 ID}}-json.log` 文件里，所以这个目录正是 fluentd 的搜集目标。



## 原理

### One Pod per Node

DaemonSet 是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？

这是一个典型的“控制器模型”能够处理的问题，类似 ReplicaSet 扩展 Pod 的数量。DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 `name=fluentdelasticsearch` 标签的 Pod 在运行。

检查的结果，可能有这三种情况：

1. 没有这种 Pod，那么就意味着要在这个 Node 上创建这样一个 Pod
2. 有这种 Pod，但是数量大于 1，那就通过 Kubernetes API 把多余的 Pod 从这个 Node 上删除掉
3. 正好只有一个这种 Pod，那说明这个节点是正常的。

要想在指定的 Node 上创建新 Pod ，可以用 `nodeSelector`，选择 Node 的名字即可。不过，在 Kubernetes 项目里，`nodeSelector `已经是一个将要被废弃的字段了。因为，现在有了一个新的、功能更完善的字段可以代替它，即：`nodeAffinity`：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In # Equal...
            values:
            - node-1
            - node-2
```

这个 Pod 里声明了一个 `spec.affnity `字段，这是 Pod 里跟调度相关的一个字段。然后定义了一个 `nodeAffinity`，它的含义是：

1. `requiredDuringSchedulingIgnoredDuringExecution`：这个 `nodeAffinity `必须在每次调度的时候予以考虑。同时，这也意味着可以设置在某些情况下不考虑这个 `nodeAffinity`
2.  这个 Pod，将来只允许运行在`metadata.name`是`node-1`或`node-2`的节点上。

DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 `nodeAffinity `定义。其中，需要绑定的节点名字，正是当前正在遍历的这个Node。

当然，DaemonSet 并不需要修改用户提交的 YAML 文件里的 Pod 模板，而是在向Kubernetes 发起请求之前，直接修改根据模板生成的 Pod 对象。



### Toleration

此外，DaemonSet 还会给这个 Pod 自动加上另外一个与调度相关的字段，叫作 `tolerations`。这个字段意味着这个 Pod，会“容忍”（Toleration）某些 Node 的“污点”（Taint）。 DaemonSet 自动加上的 `tolerations `字段，格式如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
```

这个 Toleration 的含义是：“容忍”所有被标记为 `unschedulable`“污点”的 Node；“容忍”的效果是允许调度。而在正常情况下，被标记了 `unschedulable`“污点”的 Node，是不会有任何 Pod 被调度上去的（`effect: NoSchedule`）。可是，DaemonSet 自动地给被管理的 Pod 加上了这个特殊的 Toleration，就使得这些 Pod 可以忽略这个限制，继而保证每个节点上都会被调度一个 Pod。

当然，如果这个节点有故障的话，这个 Pod 可能会启动失败，而 DaemonSet 则会始终尝试下去，直到 Pod 启动成功。



DaemonSet 的“过人之处”，就是依靠 Toleration 实现的。

假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板加上一个能够“容忍”`node.kubernetes.io/network-unavailable` “污点”的 Toleration:

```
...
template:
  metadata:
    labels:
      name: network-plugin-agent
  spec:
    tolerations:
    - key: node.kubernetes.io/network-unavailable
      operator: Exists
      effect: NoSchedule
```

在 Kubernetes 项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为`node.kubernetes.io/network-unavailable`的“污点”。

而通过这样一个 Toleration，调度器在调度这个 Pod 的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的 Agent 组件调度到这台机器上启动起来。

这种机制，正是我们在部署 Kubernetes 集群的时候，能够先部署 Kubernetes 本身、再部署网络插件的根本原因：因为当时所创建的 Flannel Pods 的 YAML，实际上就是一个 DaemonSet。



### 总结

DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

只不过，在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 `nodeAffinity`，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 `unschedulable`“污点”。

也可以在Pod 模板里加上更多种类的 Toleration，例如为了收集master 节点的日志，加入这个 Toleration：

```
tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
```



## 运行

创建这个 DaemonSet 对象：

```
$ kubectl create -f daemonSet.yaml
daemonset.apps/fluentd-elasticsearch created
```

需要注意的是，在 DaemonSet 上，一般都应该加上 resources 字段，来限制它的 CPU 和内存使用，防止它占用过多的宿主机资源。

创建成功后就能看到，如果有 N 个节点，就会有 N 个 fluentd-elasticsearch Pod 在运行：

```
$ kubectl get po -n kube-system -l name=fluentd-elasticsearch
NAME                          READY   STATUS    RESTARTS   AGE
fluentd-elasticsearch-z8nzt   1/1     Running   0          12s
fluentd-elasticsearch-zmll2   1/1     Running   0          12s
fluentd-elasticsearch-zzd4z   1/1     Running   0          12s

$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   3         3         3       3            3           <none>          21s
```

 DaemonSet 和 Deployment 一样，也有 `DESIRED`、`CURRENT `等多个状态字段。DaemonSet 也可以像 Deployment 那样，进行版本管理。这个版本，可以使用 `kubectl rollout history` 看到：

```
$ kubectl rollout history ds fluentd-elasticsearch -n kube-system
daemonset.apps/fluentd-elasticsearch
REVISION  CHANGE-CAUSE
1         <none>

# 带 record 参数更新容器镜像，第一个fluentd-elasticsearch是ds的名字，第二个是容器的名字
$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n kube-system
daemonset.apps/fluentd-elasticsearch image updated

$ kubectl rollout history ds fluentd-elasticsearch -n kube-system
daemonset.apps/fluentd-elasticsearch
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
```

Deployment 管理这些版本，靠的是：“一个版本对应一个 ReplicaSet 对象”。可是，DaemonSet 控制器操作的直接
就是 Pod，不可能有 ReplicaSet 这样的对象参与其中。那么 DaemonSet 是怎么维护版本的？

在 Kubernetes 中，一切皆对象，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现，“版本”也不例外。

**Kubernetes 中有一个名叫`ControllerRevision` 的API 对象，专门用来记录某种 Controller 对象的版本**。可以通过如下命令查看 fluentd-elasticsearch 对应的 ControllerRevision：

```
$ kubectl get controllerrevisions -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-6c85f7f4d6   daemonset.apps/fluentd-elasticsearch   2          6m41s
fluentd-elasticsearch-7f875bbfb5   daemonset.apps/fluentd-elasticsearch   1          11m

$ kubectl describe controllerrevisions -n kube-system fluentd-elasticsearch-6c85f7f4d6
Name:         fluentd-elasticsearch-6c85f7f4d6
Namespace:    kube-system
Labels:       controller-revision-hash=6c85f7f4d6
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation: 2
              kubernetes.io/change-cause:
                kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $patch:  replace
      Metadata:
        Creation Timestamp:  <nil>
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
......    
Revision:                  2
Events:                    <none>
```

可以发现，这个 ControllerRevision 对象，在 `Data `字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 `Annotation `字段保存了创建这个对象所使用的 `kubectl`命令。



对这个daemonSet 进行回滚：

```
$ kubectl rollout undo ds  fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.apps/fluentd-elasticsearch rolled back

$ kubectl get controllerrevisions -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-6c85f7f4d6   daemonset.apps/fluentd-elasticsearch   2          10m
fluentd-elasticsearch-7f875bbfb5   daemonset.apps/fluentd-elasticsearch   3          14m
```

这个操作，实际上相当于读取到了 `Revision=1 `的 ControllerRevision 对象保存的 `Data `字段。而这个 `Data `字段里保存的信息，就是 `Revision=1` 时这个 DaemonSet 的完整 API 对象。

所以，现在 DaemonSet Controller 就可以使用这个历史 API 对象，对现有的 DaemonSet 做一次 PATCH 操作（等价于执行一次 `kubectl apply -f “旧的 DaemonSet 对象”`），从而把这个 DaemonSet“更新”到一个旧版本。

在执行完这次回滚完成后，DaemonSet 的 Revision 并不会从`Revision=2` 退回到 1，而是会增加成 `Revision=3`。这是因为，一个新的 ControllerRevision被创建了出来。

> 在 Kubernetes 里，ControllerRevision 是一个通用的版本管理对象。这样， Kubernetes 项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。
>
> 不过，Deployment 是使用 ReplicaSet 进行版本管理。



