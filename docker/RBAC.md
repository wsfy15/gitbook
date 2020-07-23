# RBAC

Kubernetes 中所有的 API 对象，都保存在 Etcd 里。对这些 API 对象的操作，却一定都是通过访问 kube-apiserver 实现的。其中一个非常重要的原因，就是需要APIServer 来做**授权**工作。

在 Kubernetes 项目中，负责完成授权（Authorization）工作的机制，就是 RBAC：基于角色的访问控制（Role-Based Access Control）。

这个机制有三个最基本的概念：

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的**操作权限**。
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以是你在 Kubernetes 里定义的“用户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。



## Role

Role 本身就是一个 Kubernetes 的 API 对象：

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
 - apiGroups: [""]
   resources: ["pods"]
   verbs: ["get", "watch", "list"]
```

这个 Role 对象指定了它能产生作用的 Namepace 是：`mynamespace`。Namespace 是 Kubernetes 中一个逻辑管理单位，不同 Namespace 的 API 对象，在通过 `kubectl `命令进行操作的时候，是互相隔离开的。

当然，这仅限于逻辑上的“隔离”，Namespace 并不会提供任何实际的隔离或者多租户能力。没有指定 Namespace的情况下，就是使用默认的Namespace：`default`。

这个 Role 对象的 `rules `字段，就是它所定义的权限规则。在上面的例子里，这条规则的含义就是：允许“被作用者”，对 `mynamespace `下面的 Pod 对象，进行 GET、WATCH 和 LIST 操作。

> verbs：create、get、list、watch 、update、patch、delete 

`rules`还可以进一步细化，比如可以只针对某一个具体的对象进行权限设置：

```
rules:
 - apiGroups: [""]
   resources: ["configmaps"]
   resourceNames: ["my-config"]
   verbs: ["get"]
```

这条规则的“被作用者”，只对名叫`my-config`的 ConfigMap 对象，有进行 GET 操作的权限。





## RoleBinding

为了将一个Role 作用于 “被作用者”上，需要通过 RoleBinding 实现。RoleBinding 也是一个 Kubernetes 的 API 对象：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

 `subjects `字段就是“被作用者”。它的类型是 `User`，即 Kubernetes 里的用户。这个用户的名字是 example-user。

不过，在 Kubernetes 中，并没有一个叫作“User”的 API 对象。实际上，Kubernetes 里的“User”，也就是“用户”，只是一个授权系统里的逻辑概念。它需要通过外部认证服务来提供，比如 Keystone。或者，直接给 APIServer 指定一个用户名、密码文件，Kubernetes 的授权系统，就能够从这个文件里找到对应的“用户”了。当然，在大多数私有的使用环境中，我们只要使用 Kubernetes 提供的内置“用户”，就足够了。



`roleRef `字段允许 RoleBinding 对象直接通过名字，引用前面定义的 Role 对象，从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系。



Role 和 RoleBinding 对象都是 Namespaced 对象（Namespaced Object），它们对权限的限制规则仅在它们自己的 Namespace 内有效，roleRef 也只能引用当前 Namespace 里的 Role 对象。

对于非 Namespaced（Non-namespaced）对象（比如：Node），或者，某一个Role 想要作用于所有的 Namespace 的时候，可以使用 ClusterRole 和 ClusterRoleBinding 这两个组合了。这两个 API 对象的用法跟 Role 和 RoleBinding 完全一样。只不过，它们的定义里，没有了 Namespace 字段：

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

example-user 用户，拥有对所有 Namespace 里的 Pod 进行 GET、WATCH 和 LIST 操作的权限。



## 内置用户

在大多数时候，我们其实都不太使用“用户”这个功能，而是直接使用Kubernetes 里的“内置用户”。这个由 Kubernetes 负责管理的“内置用户”，就是ServiceAccount。ServiceAccount 分配权限的过程如下：

- 首先，定义一个 ServiceAccount。它的 API 对象非常简单：

  ```
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    namespace: mynamespace
    name: example-sa
  ```

  一个最简单的 ServiceAccount 对象只需要 `Name `和 `Namespace `这两个最基本的字段。

- 然后编写 RoleBinding 的 YAML 文件，来为这个 ServiceAccount 分配权限：

  ```
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: example-rolebinding
    namespace: mynamespace
  subjects:
  - kind: ServiceAccount
    name: example-sa
    namespace: mynamespace
  roleRef:
    kind: Role
    name: example-role
    apiGroup: rbac.authorization.k8s.io
  ```

- 创建Role、SA、RoleBinding对象：

  ```
  $ kubectl create namespace mynamespace
  $ kubectl create -f svc-account.yaml
  $ kubectl create -f role.yaml
  $ kubectl create -f role-binding.yaml
  ```

- 查看这个 ServiceAccount 的详细信息：

  ```
  $ kubectl get sa -n mynamespace -o yaml
  ...
  - apiVersion: v1
    kind: ServiceAccount
    metadata:
      creationTimestamp: "2020-07-18T09:51:08Z"
      name: example-sa
      namespace: mynamespace
      resourceVersion: "640500"
      selfLink: /api/v1/namespaces/mynamespace/serviceaccounts/example-sa
      uid: 61600045-c0ad-46fa-bb04-55b9c6f8b12f
    secrets:
    - name: example-sa-token-58szm
  ...
  ```

  Kubernetes 会为一个 ServiceAccount 自动创建并分配一个 Secret 对象，即：上述 ServiceAcount 定义里最下面的 secrets 字段。
  这个 Secret，就是这个 ServiceAccount 对应的、用来跟 APIServer 进行交互的授权文件，一般称它为：Token。Token 文件的内容一般是证书或者密码，它以一个 Secret 对象的方式保存在 Etcd 当中。

- 这样，用户的 Pod，就可以声明使用这个 ServiceAccount 了：

  ```
  apiVersion: v1
  kind: Pod
  metadata:
    namespace: mynamespace
    name: sa-token-test
  spec:
    containers:
    - name: nginx
      image: nginx:1.17.8
    serviceAccountName: example-sa
  ```

  ```
  $ kubectl create -f nginx-pod.yaml
  $ kubectl describe po sa-token-test -n mynamespace
  Name:         sa-token-test
  Namespace:    mynamespace
  ...
  Containers:
    nginx:
      ...
      Mounts:
        /var/run/secrets/kubernetes.io/serviceaccount from example-sa-token-58szm (ro)
  ```

  该 ServiceAccount 的 token，也就是一个Secret 对象，被 Kubernetes 自动挂载到了容器的`/var/run/secrets/kubernetes.io/serviceaccount` 目录下，容器里的应用，可以使用这个目录底下的 `ca.crt `来访问 APIServer 。更重要的是，此时它只能够做 GET、WATCH 和 LIST 操作。因为 example-sa 这个 ServiceAccount 的权限，已经被绑定了 Role 做了限制。

如果一个Pod 没有声明 serviceAccountName，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。

但在这种情况下，这个默认 ServiceAccount 并没有关联任何 Role。也就是说，此时**它有访问APIServer 的绝大多数权限。**当然，这个访问所需要的 Token，还是默认 ServiceAccount 对应的 Secret 对象为它提供的，如下所示。

 ```
 $ kubectl describe sa
 Name:                default
 Namespace:           default
 Labels:              <none>
 Annotations:         <none>
 Image pull secrets:  <none>
 Mountable secrets:   default-token-l489d
 Tokens:              default-token-l489d
 Events:              <none>
 
 $ kubectl get secret
 NAME                  TYPE                                  DATA   AGE
 default-token-l489d   kubernetes.io/service-account-token   3      14d
 
 $ kubectl describe secret default-token-l489d
 Name:         default-token-l489d
 Namespace:    default
 Labels:       <none>
 Annotations:  kubernetes.io/service-account.name: default
               kubernetes.io/service-account.uid: 005219b8-3554-4e68-898c-dec11f099c3a
 
 Type:  kubernetes.io/service-account-token
 
 Data
 ......
 ca.crt:     1025 bytes
 namespace:  7 bytes
 ```

 Kubernetes 会自动为默认 ServiceAccount 创建并绑定一个特殊的 Secret：它的类型是`kubernetes.io/service-account-token`；它的 Annotation 字段，声明了`kubernetes.io/service-account.name=default`，即这个 Secret 会跟同一Namespace 下名叫 default 的 ServiceAccount 进行绑定。

> 在生产环境中，强烈建议为所有 Namespace 下的默认 ServiceAccount，绑定一个只读权限的 Role。



## 用户组

除了前面使用的“用户”（User），Kubernetes 还拥有“用户组”（Group）的概念，也就是一组“用户”的意思。如果为 Kubernetes 配置了外部认证服务的话，这个“用户组”的概念就会由外部认证服务提供。

而对于 Kubernetes 的内置“用户”ServiceAccount 来说，上述“用户组”的概念也同样适用。实际上，一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：`system:serviceaccount:<ServiceAccount 名字 >`。它对应的内置“用户组”的名字，就是：`system:serviceaccounts:<Namespace 名字 >`。

在 RoleBinding 里定义如下的 subjects，就意味着这个 Role 的权限规则，作用于 mynamespace 里的所有 ServiceAccount。这就用到了“用户组”的概念。

```
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```



在 Kubernetes 中已经内置了很多个为系统保留的 ClusterRole，它们的名字都以 `system: `开头，可以通过 `kubectl get clusterroles` 查看到它们。一般来说，这些系统 ClusterRole，是绑定给 Kubernetes 系统组件对应的 ServiceAccount 使用的。

比如，其中一个名叫 `system:kube-scheduler` 的 ClusterRole，定义的权限规则是 kube scheduler（Kubernetes 的调度器组件）运行所需要的必要权限：

```
$ kubectl describe clusterrole system:kube-scheduler
Name:         system:kube-scheduler
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
PolicyRule:
  Resources                                  Non-Resource URLs  Resource Names    Verbs
  ---------                                  -----------------  --------------    -----
  events                                     []                 []                [create patch update]
  events.events.k8s.io                       []                 []                [create patch update]
  bindings                                   []                 []                [create]
  endpoints                                  []                 []                [create]
  pods/binding                               []                 []                [create]
  tokenreviews.authentication.k8s.io         []                 []                [create]
  subjectaccessreviews.authorization.k8s.io  []                 []                [create]
  leases.coordination.k8s.io                 []                 []                [create]
  pods                                       []                 []                [delete get list watch]
  nodes                                      []                 []                [get list watch]
  persistentvolumeclaims                     []                 []                [get list watch]
  persistentvolumes                          []                 []                [get list watch]
  replicationcontrollers                     []                 []                [get list watch]
  services                                   []                 []                [get list watch]
  replicasets.apps                           []                 []                [get list watch]
  statefulsets.apps                          []                 []                [get list watch]
  replicasets.extensions                     []                 []                [get list watch]
  poddisruptionbudgets.policy                []                 []                [get list watch]
  csinodes.storage.k8s.io                    []                 []                [get list watch]
  endpoints                                  []                 [kube-scheduler]  [get update]
  leases.coordination.k8s.io                 []                 [kube-scheduler]  [get update]
  pods/status                                []                 []                [patch update]
```

这个 `system:kube-scheduler` 的 ClusterRole，就会被绑定给 kube-system Namesapce 下名叫 `kube-scheduler `的 ServiceAccount，它正是 Kubernetes 调度器的 Pod 声明使用的ServiceAccount。



除此之外，Kubernetes 还提供了四个预先定义好的 ClusterRole 来供用户直接使用：

1. cluster-amdin：整个 Kubernetes 中的最高权限(`verbs=*`)
2. admin
3. edit
4. view：只有 Kubernetes API 的只读权限





