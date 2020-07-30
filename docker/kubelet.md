# kubelet

在调度这一步完成后，Kubernetes 就需要负责将这个调度成功的 Pod，在宿主机上创建出来，并把它所定义的各个容器启动起来。这些，都是 kubelet 这个核心组件的主要功能。



## 工作原理

![工作原理](kubelet.assets/1596002707575.png)

 kubelet 也是按照“控制器”模式来工作的，它的工作核心，就是一个控制循环，即：SyncLoop。而驱动这个控制循环运行的事件，包括四种：

1. Pod 更新事件；
2. Pod 生命周期变化；
3. kubelet 本身设置的执行周期；
4. 定时的清理事件。



跟其他控制器类似，kubelet 启动的时候，要做的第一件事情，就是设置 `Listers`，也就是注册它所关心的各种事件的 Informer。这些 Informer，就是 SyncLoop 需要处理的数据的来源。

kubelet 还负责维护着很多很多其他的子控制循环，这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager 等等。这些控制循环的责任，就是通过控制器模式，完成 kubelet 的某项具体职责。比如 Node Status Manager，就负责响应 Node 的状态变化，然后将 Node 的状态收集起来，并通过 Heartbeat 的方式上报给 APIServer。再比如 CPU Manager，就负责维护该 Node 的 CPU 核的信息，以便在 Pod 通过 cpuset 的方式请求 CPU 核的时候，能够正确地管理 CPU 核的使用量和可用量。



**SyncLoop，又是如何根据 Pod 对象的变化，来进行容器操作的呢？**

实际上，kubelet 也是通过 Watch 机制，监听了与自己相关的 Pod 对象的变化。当然，这个Watch 的过滤条件是该 Pod 的 `nodeName `字段与自己相同。kubelet 会把这些 Pod 的信息缓存在自己的内存里。

当一个 Pod 完成调度、与一个 Node 绑定起来之后， 这个 Pod 的变化就会触发 kubelet 在控制循环里注册的 Handler，也就是图中的 HandlePods 部分。此时，通过检查该 Pod 在 kubelet 内存里的状态，kubelet 就能够判断出这是一个新调度过来的 Pod，从而触发 Handler里 ADD 事件对应的处理逻辑。

在具体的处理过程当中，kubelet 会启动一个名叫 Pod Update Worker 的、单独的Goroutine 来完成对 Pod 的处理工作。

比如，如果是 ADD 事件的话，kubelet 就会为这个新的 Pod 生成对应的 Pod Status，检查Pod 所声明使用的 Volume 是不是已经准备好。然后，调用下层的容器运行时（比如Docker），开始创建这个 Pod 所定义的容器。

如果是 UPDATE 事件的话，kubelet 就会根据 Pod 对象具体的变更情况，调用下层容器运行时进行容器的重建工作。



## CRI

kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker 的 API，而是通过一组叫作 **CRI**（Container Runtime Interface，容器运行时接口）的 gRPC接口来间接执行的。

Kubernetes 项目之所以要在 kubelet 中引入这样一层单独的抽象，是为了对 Kubernetes屏蔽下层容器运行时的差异。因为有很多种容器的实现，例如 Docker、rkt、runV等等，如果不引入接口层，而是直接操作相应的容器实现，势必会导致kubelet 代码量的膨胀。

引入CRI 后，Kubernetes 以及 kubelet 本身的架构如图所示：

![kubelet 在kubernetes中的架构](kubelet.assets/1596003345663.png)

当 Kubernetes 通过编排能力创建了一个 Pod 之后，调度器会为这个 Pod 选择一个具体的节点来运行。这时候，kubelet 就会通过 SyncLoop 来判断需要执行的具体操作，比如创建一个 Pod。此时，kubelet 就会调用一个叫作`GenericRuntime `的通用组件来发起创建 Pod 的 CRI 请求。

如果你使用的容器实现是 Docker 的话，那么负责响应这个请求的就是一个叫作 dockershim 的组件。它会把 CRI 请求里的内容拿出来，然后组装成 Docker API 请求发给 Docker Daemon。

而更普遍的场景，就是需要在每台宿主机上单独安装一个负责响应 CRI 的组件，这个组件，一般被称作 CRI shim。CRI shim 的工作，就是扮演 kubelet 与容器实现之间的“垫片”（shim）。所以它的作用非常单一，那就是实现 CRI 规定的每个接口，然后把具体的 CRI 请求“翻译”成对后端容器实现的请求或者操作。



CRI 机制能够发挥作用的核心，就在于每一种容器实现都可以自己实现一个 CRI shim，自行对 CRI 请求进行处理。这样，Kubernetes 就有了一个统一的容器抽象层，使得下层容器运行时可以自由地对接进入 Kubernetes 当中。

这里的 CRI shim，就是容器项目的维护者们自由发挥的“场地”了。除了 dockershim 之外，其他容器运行时的 CRI shim，都是需要额外部署在宿主机上的。例如 CNCF 里的 containerd 项目，就可以提供一个典型的 CRI shim 的能力，即：将Kubernetes 发出的 CRI 请求，转换成对 containerd 的调用，然后创建出 runC 容器。而 runC 项目，才是负责执行设置容器 Namespace、Cgroups 和 chroot 等基础操作的组件。

![kubelet、CRI、容器实现 间关系](kubelet.assets/1596003951476.png)



### 主要接口

可以把 CRI 里主要的[接口](https://godoc.org/k8s.io/cri-api/pkg/apis/runtime/v1alpha2)分为两组：

- **RuntimeService**，它提供的接口，主要是跟容器相关的操作。比如，创建和启动容器、删除容器、执行 exec 命令等等。

  ![Runtime接口](kubelet.assets/1596004844574.png)

- **ImageService**，它提供的接口，主要是容器镜像相关的操作，比如拉取镜像、删除镜像等等。

  ![镜像接口](kubelet.assets/1596004910729.png)

在RuntimeService 中，CRI 设计的一个重要原则，就是确保这个接口本身，**只关注容器，不关注 Pod**。因为Pod 是 Kubernetes 的编排概念，而不是容器运行时的概念。所以不能假设所有下层容器项目，都能够暴露出可以直接映射为 Pod 的 API。而且，如果 CRI 里引入了关于 Pod 的概念，那么只要 Pod API 对象的字段发生变化，那么 CRI 就很有可能需要变更。

所以，在 CRI 的设计里，并没有一个直接创建 Pod 或者启动 Pod 的接口。

不过，CRI 里还是有一组叫作 `RunPodSandbox `的接口的。这个 PodSandbox，对应的并不是 Kubernetes 里的 Pod API 对象，而只是抽取了 Pod 里的一部分与容器运行时相关的字段，比如 `HostName`、`DnsConfig`、`CgroupParent `等。所以PodSandbox 这个接口描述的，其实是 Kubernetes 将 Pod 这个概念映射到容器运行时层面所需要的字段，或者说是一个 Pod 对象子集。

对于具体的容器项目，可以自己决定如何使用这些字段来实现一个 Kubernetes 期望的Pod 模型。

![CRI 创建Pod的原理](kubelet.assets/1596005057152.png)

比如，在执行 `kubectl run` 创建了一个名叫 foo 的、包括了 A、B 两个容器的 Pod 之后。这个 Pod 的信息最后来到 kubelet，kubelet 就会按照图中所示的顺序来调用 CRI 接口。

在具体的 CRI shim 中，这些接口的实现是可以完全不同的。如果是 Docker 项目，dockershim 就会创建出一个名叫 foo 的 Infra 容器（pause 容器），用来“hold”住整个Pod 的 Network Namespace。而如果是基于虚拟化技术的容器，比如 Kata Containers 项目，它的 CRI 实现就会直接创建出一个轻量级虚拟机来充当 Pod。

> 在 `RunPodSandbox `这个接口的实现中，还需要调用`networkPlugin.SetUpPod(…) `来为这个 Sandbox 设置网络。这个 `SetUpPod(…)` 方法，实际上就在执行 CNI 插件里的` add(…) `方法，也就是 CNI 插件为 Pod 创建网络，并且把 Infra 容器加入到网络中的操作。



接下来，kubelet 继续调用 `CreateContainer `和 `StartContainer `接口来创建和启动容器 A、B。对应到 dockershim 里，就是直接启动 A，B 两个 Docker 容器。所以最后，宿主机上会出现三个 Docker 容器组成这一个 Pod。

而如果是 Kata Containers 的话，`CreateContainer `和 `StartContainer `接口的实现，就只会在前面创建的轻量级虚拟机里创建两个 A、B 容器对应的 Mount Namespace。所以，最后在宿主机上，只会有一个叫作 foo 的轻量级虚拟机在运行。



### Streaming API

除了对容器生命周期的实现之外，CRI shim 还有一个重要的工作，就是如何实现 exec、logs 等接口。这些接口跟前面的操作有一个很大的不同，就是这些 gRPC 接口调用期间，kubelet 需要跟容器项目**维护一个长连接来传输数据**。这种 API，称之为 Streaming API。

CRI shim 里对 Streaming API 的实现，依赖于一套独立的 Streaming Server 机制。

![Streaming API原理](kubelet.assets/1596005444628.png)

当我们对一个容器执行 `kubectl exec` 命令的时候，这个请求首先交给 API Server，然后 API Server 就会调用 kubelet 的 Exec API。这时，kubelet 就会调用 CRI 的 Exec 接口，而负责响应这个接口的，自然就是具体的 CRI shim。

但在这一步，CRI shim 并不会直接去调用后端的容器项目（比如 Docker ）来进行处理，而只会返回一个 URL 给 kubelet。这个 URL，就是该 CRI shim 对应的 Streaming Server 的地址和端口。

而 kubelet 在拿到这个 URL 之后，就会把它以 Redirect 的方式返回给 API Server。所以这时候，API Server 就会通过重定向来向 Streaming Server 发起真正的` /exec` 请求，与它建立长连接。

这个 Streaming Server 本身，是需要通过使用 SIG-Node 维护的 Streaming API库来实现的。并且，Streaming Server 会在 CRI shim 启动时就一起启动。此外，Streaming Server 这一部分具体怎么实现，完全可以由 CRI shim 的维护者自行决定。比如，对于 Docker 来说，dockershim 就是直接调用 Docker 的 Exec API 来作为实现的。



## 安全容器

所谓安全容器，就是给进程分配一个**独立的操作系统内核**，从而避免了让容器共享宿主机的内核。这样，容器进程能够看到的攻击面，就从整个宿主机内核变成了一个极小的、独立的、以容器为单位的内核，从而有效解决了容器进程发生“逃逸”或者夺取整个宿主机的控制权的问题。

![安全容器原理](kubelet.assets/1596076478666.png)



主要有两种类型的安全容器：

- **Kata Containers** ，使用传统的虚拟化技术，通过虚拟硬件模拟出了一台“小虚拟机”，然后在这个小虚拟机里安装了一个裁剪后的 Linux 内核来实现**强隔离**。
- **gVisor**，直接用 Go 语言“模拟”出了一个运行在用户态的操作系统内核，然后通过这个模拟的内核来代替容器进程向宿主机发起有限的、可控的系统调用。



### KataContainers

![KataContainers 工作原理](kubelet.assets/1596076988701.png)

Kata Containers 的本质，就是一个轻量化虚拟机。所以启动一个 Kata Containers 之后，其实就是一个正常的虚拟机在运行。这也就意味着，一个标准的虚拟机管理程序（Virtual Machine Manager, VMM）是运行 Kata Containers 必备的一个组件。在上图中，使用的 VMM 就是 Qemu。

使用虚拟机作为进程的隔离环境，Kata Containers 原生就带有了 Pod 的概念。即：这个 Kata Containers 启动的虚拟机，就是一个 Pod；而用户定义的容器，就是运行在这个轻量级虚拟机里的进程。

在具体实现上，Kata Containers 的虚拟机里会有一个特殊的 Init 进程负责管理虚拟机里面的用户容器，并且只为这些容器开启 Mount Namespace。所以，这些用户容器之间，原生就是共享 Network 以及其他 Namespace 的。

此外，为了跟上层编排框架比如 Kubernetes 进行对接，Kata Containers 项目会启动一系列跟用户容器对应的 shim 进程，来负责操作这些用户容器的生命周期。当然，这些操作，实际上还是要靠虚拟机里的 Init 进程做到。

在具体的架构上，Kata Containers 的实现方式同一个正常的虚拟机其实也非常类似。

![Kata Containers 架构](kubelet.assets/1596078688635.png)

当 Kata Containers 运行起来之后，虚拟机里的用户进程（容器），实际上只能看到虚拟机里的、被裁减过的 Guest Kernel，以及通过 Hypervisor 虚拟出来的硬件设备。

为了能够对这个虚拟机的 I/O 性能进行优化，Kata Containers 也会通过 vhost 技术（比如：vhost-user）来实现 Guest 与 Host 之间的高效的网络通信，并且使用 PCI Passthrough（PCI 穿透）技术来让 Guest 里的进程直接访问到宿主机上的物理设备。这些架构设计与实现，跟常规虚拟机的优化手段是基本一致的。



### gVisor

![gVisor 工作原理](kubelet.assets/1596081015320.png)

gVisor 工作的核心，在于它为应用进程、也就是用户容器，启动了一个名叫 Sentry 的进程。而 Sentry 进程的主要职责，就是提供一个传统的操作系统内核的能力，即：运行用户程序，执行系统调用。所以说，Sentry 并不是使用 Go 语言重新实现了一个完整的 Linux 内核，而只是一个**对应用进程“冒充”内核**的系统组件。

在这种设计思想下，Sentry 需要自己实现一个完整的 Linux 内核网络栈，以便处理应用进程的通信请求。然后，把封装好的二层帧直接发送给 Kubernetes 设置的 Pod 的 Network Namespace 即可。此外，Sentry 对于 Volume 的操作，则需要通过 9p 协议交给一个叫做 Gofer 的代理进程来完成。Gofer 会代替应用进程直接操作宿主机上的文件，并依靠 seccomp 机制将自己的能力限制在最小集，从而防止恶意应用进程通过 Gofer 来从容器中“逃逸”出去。

在具体的实现上，gVisor 的 Sentry 进程，还分为两种不同的实现方式：

- 使用 Ptrace 机制拦截用户应用的系统调用（System Call），然后把这些系统调用交给 Sentry 来进行处理。

  ![Ptrace 拦截系统调用](kubelet.assets/1596081131245.png)

  这个过程对于应用进程来说，是完全透明的。而 Sentry 接下来，则会扮演操作系统的角色，在用户态执行用户程序，然后仅在需要的时候，才向宿主机发起 Sentry 自己所需要执行的系统调用。这就是 gVisor 对用户应用进程进行强隔离的主要手段。不过， Ptrace 进行系统调用拦截的性能实在是太差，仅能供 Demo 时使用。
  
- 使用 KVM 来进行系统调用的拦截，这个性能比 Ptrace 就要好很多了。

  为了能够做到这一点，Sentry 进程就必须扮演一个 Guest Kernel 的角色，负责执行用户程序，发起系统调用。而这些系统调用被 KVM 拦截下来，还是继续交给 Sentry 进行处理。只不过在这时候，Sentry 就切换成了一个普通的宿主机进程的角色，来向宿主机发起它所需要的系统调用。

  ![1596081260723](kubelet.assets/1596081260723.png)

  **在这种实现里，Sentry 并不会真的像虚拟机那样去虚拟出硬件设备、安装 Guest 操作系统。它只是借助 KVM 进行系统调用的拦截，以及处理地址空间切换等细节。**



### 对比

在性能上，KataContainers 和 KVM 实现的 gVisor 基本不分伯仲，在启动速度和占用资源上，基于用户态内核的 gVisor 还略胜一筹。但是，对于系统调用密集的应用，比如重 I/O 或者重网络的应用，gVisor 就会因为需要频繁拦截系统调用而出现性能急剧下降的情况。此外，gVisor 由于要自己使用 Sentry 去模拟一个 Linux 内核，所以它能支持的系统调用是有限的，只是 Linux 系统调用的一个子集。

